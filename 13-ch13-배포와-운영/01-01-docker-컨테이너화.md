# Docker 컨테이너화

> MCP 서버를 Docker 컨테이너로 패키징하고, 멀티 스테이지 빌드와 Docker Compose로 프로덕션 배포를 준비하는 방법을 배웁니다.

## 개요

이 섹션에서는 MCP 서버를 Docker 컨테이너로 패키징하는 전체 과정을 다룹니다. 지금까지 우리는 로컬 환경에서 `python server.py`로 MCP 서버를 실행해 왔는데요, 프로덕션에서는 환경 차이, 의존성 충돌, 보안 격리 등 수많은 문제가 기다리고 있습니다. Docker는 이 모든 문제를 한 번에 해결하는 표준 도구입니다.

**선수 지식**: [MCP 서버 개발 기초](03-ch3-개발-환경-설정과-첫-mcp-서버/02-02-hello-mcp-첫-번째-서버-만들기.md), [Streamable HTTP Transport](02-ch2-mcp-아키텍처와-프로토콜-구조/03-03-transport-계층-streamable-http.md), [에러 처리와 로깅](12-ch12-에러-처리-로깅-테스트/02-02-구조화된-로깅과-모니터링.md) 개념

**학습 목표**:
- MCP 서버용 Dockerfile을 작성하고 멀티 스테이지 빌드로 이미지를 최적화할 수 있다
- 환경 변수를 통해 설정을 안전하게 주입할 수 있다
- Docker Compose로 MCP 서버 + 데이터베이스 다중 서비스를 구성할 수 있다
- Docker MCP Toolkit/Catalog의 역할을 이해하고 활용할 수 있다
- non-root 사용자, 최소 이미지 등 보안 모범 사례를 적용할 수 있다

## 왜 알아야 할까?

"내 컴퓨터에서는 잘 되는데요?" — 개발자라면 한 번쯤 이 말을 해 봤거나 들어 봤을 겁니다. MCP 서버도 예외가 아닙니다.

[Ch3에서 만든 첫 서버](03-ch3-개발-환경-설정과-첫-mcp-서버/02-02-hello-mcp-첫-번째-서버-만들기.md)를 동료에게 전달하려면 어떻게 해야 할까요? Python 버전 맞추기, 가상 환경 세팅, 의존성 설치… 단계가 끝이 없죠. 하지만 Docker가 있으면 `docker run -i mcp-server` 한 줄로 끝납니다.

MCP 서버의 Docker 컨테이너화가 특히 중요한 이유가 있습니다:

1. **보안 격리**: MCP 서버는 LLM이 호출하는 도구를 제공하므로, 잘못된 입력이 호스트 시스템을 위협할 수 있습니다. 컨테이너는 이 위험을 샌드박스 안에 가둡니다. [Ch11 인증과 보안](11-ch11-인증과-보안/01-01-mcp-보안-모델과-위협-분석.md)에서 다룬 보안 위협 — 프롬프트 인젝션, 도구 남용, 데이터 유출 등 — 을 Docker의 격리 계층이 추가적으로 방어해 줍니다.
2. **Transport 호환성**: stdio Transport는 stdin/stdout 버퍼링 문제가 까다로운데, Docker 환경에서 이를 일관되게 제어할 수 있습니다.
3. **배포 표준화**: Docker Hub의 MCP Catalog에는 이미 수백 개의 검증된 MCP 서버가 `mcp/` 네임스페이스로 게시되어 있습니다. 여러분의 서버도 같은 방식으로 배포할 수 있죠.

> 📊 **그림 1**: 로컬 실행 vs Docker 컨테이너 실행 비교

```mermaid
flowchart LR
    subgraph LOCAL["로컬 실행"]
        A1["Python 3.12 설치"] --> A2["venv 생성"]
        A2 --> A3["pip install"]
        A3 --> A4["환경변수 설정"]
        A4 --> A5["python server.py"]
    end

    subgraph DOCKER["Docker 실행"]
        B1["docker run -i<br/>mcp-server"] --> B2["즉시 실행"]
    end

    LOCAL -.->|"5단계"| A5
    DOCKER -.->|"1단계"| B2

    style LOCAL fill:#FF9800,color:#fff
    style DOCKER fill:#4CAF50,color:#fff
```

## 핵심 개념

### 개념 1: MCP 서버용 Dockerfile 기초

> 💡 **비유**: Dockerfile은 요리 레시피와 같습니다. 재료(베이스 이미지)를 준비하고, 조리 도구(의존성)를 세팅하고, 조리법(코드 복사)을 따라 하고, 최종 요리(서버)를 내놓는 단계별 지시서입니다. 한 번 레시피를 적어두면, 누구든 어디서든 같은 요리를 만들 수 있죠.

MCP 서버를 컨테이너화할 때 가장 중요한 것은 **Transport 방식에 따라 Dockerfile이 달라진다**는 점입니다. [Ch2에서 배운 두 가지 Transport](02-ch2-mcp-아키텍처와-프로토콜-구조/02-02-transport-계층-stdio.md) — stdio와 Streamable HTTP — 각각의 요구사항이 다릅니다.

**stdio Transport용 Dockerfile:**

```python
# server.py — stdio MCP 서버
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("docker-demo")

@mcp.tool()
def greet(name: str) -> str:
    """사용자에게 인사합니다."""
    return f"안녕하세요, {name}님! Docker 컨테이너에서 인사드립니다."

if __name__ == "__main__":
    mcp.run()  # 기본: stdio transport
```

```dockerfile
# Dockerfile — stdio Transport용
FROM python:3.12-slim

WORKDIR /app

# 의존성 먼저 복사 (레이어 캐시 활용)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 소스 코드 복사
COPY server.py .

# 핵심! Python 출력 버퍼링 비활성화
ENV PYTHONUNBUFFERED=1

CMD ["python", "server.py"]
```

여기서 `PYTHONUNBUFFERED=1`이 **절대적으로 중요**합니다. 이 환경 변수가 없으면 Python이 stdout을 버퍼링하는데, MCP의 stdio Transport는 stdout으로 JSON-RPC 메시지를 주고받기 때문에 버퍼링이 발생하면 클라이언트가 응답을 기다리다 영원히 멈춰버립니다.

> ⚠️ **흔한 오해**: "Docker 컨테이너는 항상 데몬(백그라운드 서비스)으로 실행해야 한다"고 생각하기 쉽습니다. 하지만 stdio Transport MCP 서버는 `docker run -i`(interactive 모드)로 실행해야 합니다. `-i` 플래그가 없으면 stdin이 닫혀서 클라이언트가 요청을 보낼 수 없거든요. 반면 Streamable HTTP Transport 서버는 `-d`(detached 모드)로 실행합니다.

**Streamable HTTP Transport용 Dockerfile:**

```python
# server_http.py — Streamable HTTP MCP 서버
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("docker-http-demo")

@mcp.tool()
def calculate(expression: str) -> str:
    """수학 표현식을 계산합니다."""
    try:
        result = eval(expression, {"__builtins__": {}})  # 안전한 eval
        return f"결과: {result}"
    except Exception as e:
        return f"계산 오류: {e}"

if __name__ == "__main__":
    mcp.run(transport="streamable-http", host="0.0.0.0", port=8000)
```

```dockerfile
# Dockerfile — Streamable HTTP Transport용
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY server_http.py .

ENV PYTHONUNBUFFERED=1

# HTTP 포트 노출
EXPOSE 8000

CMD ["python", "server_http.py"]
```

> 📊 **그림 2**: stdio vs HTTP Transport의 Docker 실행 차이

```mermaid
flowchart TD
    subgraph STDIO["stdio Transport"]
        S1["docker run -i --rm"] --> S2["stdin/stdout 연결"]
        S2 --> S3["MCP Client<br/>직접 연결"]
    end

    subgraph HTTP["Streamable HTTP Transport"]
        H1["docker run -d -p 8000:8000"] --> H2["HTTP 포트 노출"]
        H2 --> H3["네트워크 통한<br/>원격 접속"]
    end

    S3 -->|"로컬 전용"| E1["1:1 프로세스"]
    H3 -->|"원격 가능"| E2["다중 클라이언트"]

    style STDIO fill:#2196F3,color:#fff
    style HTTP fill:#4CAF50,color:#fff
```

빌드와 실행은 Transport에 따라 다릅니다:

```run:python
# Transport별 Docker 명령어 비교
commands = {
    "stdio": {
        "build": "docker build -t mcp-stdio-server .",
        "run": "docker run -i --rm mcp-stdio-server",
        "특징": "stdin 연결 필수 (-i), 1:1 프로세스",
    },
    "streamable-http": {
        "build": "docker build -t mcp-http-server .",
        "run": "docker run -d -p 8000:8000 mcp-http-server",
        "특징": "포트 매핑 (-p), 백그라운드 실행 (-d)",
    },
}

for transport, info in commands.items():
    print(f"[{transport}]")
    for key, value in info.items():
        print(f"  {key}: {value}")
    print()
```

```output
[stdio]
  build: docker build -t mcp-stdio-server .
  run: docker run -i --rm mcp-stdio-server
  특징: stdin 연결 필수 (-i), 1:1 프로세스

[streamable-http]
  build: docker build -t mcp-http-server .
  run: docker run -d -p 8000:8000 mcp-http-server
  특징: 포트 매핑 (-p), 백그라운드 실행 (-d)
```

### 개념 2: 멀티 스테이지 빌드로 이미지 최적화

> 💡 **비유**: 요리를 할 때 온갖 조리 도구가 필요하지만, 손님에게 요리를 내놓을 때는 접시 위의 음식만 있으면 됩니다. 멀티 스테이지 빌드는 "주방(빌드 스테이지)"과 "식탁(런타임 스테이지)"을 분리하여, 최종 이미지에는 실행에 필요한 것만 담는 기법입니다.

단일 스테이지로 빌드하면 컴파일러, 빌드 도구, 캐시 파일 등이 최종 이미지에 그대로 남습니다. 이는 이미지 크기를 불필요하게 키우고, 공격 표면(attack surface)도 넓힙니다.

```dockerfile
# Dockerfile.multistage — 프로덕션 최적화 빌드
# ============================================

# Stage 1: 빌드 환경
FROM python:3.12-slim AS builder

WORKDIR /app

# 가상 환경 생성 — 시스템 패키지와 분리
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# 의존성 설치 (빌드 도구 포함)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# -------------------------------------------
# Stage 2: 런타임 환경 (최소 이미지)
FROM python:3.12-slim

# 빌드 스테이지에서 가상 환경만 복사
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# 보안: non-root 사용자 생성
RUN groupadd -r mcp && useradd -r -g mcp -d /app -s /sbin/nologin mcp

WORKDIR /app

# 소스 코드 복사
COPY --chown=mcp:mcp . .

# 환경 설정
ENV PYTHONUNBUFFERED=1
ENV MCP_ENV=production

# non-root 사용자로 전환
USER mcp

EXPOSE 8000

# 헬스체크 (HTTP Transport용)
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

CMD ["python", "server.py"]
```

> 📊 **그림 3**: 멀티 스테이지 빌드 과정

```mermaid
flowchart LR
    subgraph STAGE1["Stage 1: Builder"]
        B1["python:3.12-slim"] --> B2["venv 생성"]
        B2 --> B3["pip install"]
        B3 --> B4["/opt/venv<br/>(의존성 패키지)"]
    end

    subgraph STAGE2["Stage 2: Runtime"]
        R1["python:3.12-slim"] --> R2["COPY --from=builder<br/>/opt/venv"]
        R2 --> R3["non-root 사용자"]
        R3 --> R4["소스 코드 복사"]
        R4 --> R5["최종 이미지<br/>~150MB"]
    end

    B4 -->|"venv만 전달"| R2

    style STAGE1 fill:#FF9800,color:#fff
    style STAGE2 fill:#4CAF50,color:#fff
```

멀티 스테이지 빌드의 효과는 극적입니다:

```run:python
# 이미지 크기 비교 시뮬레이션
stages = {
    "단일 스테이지 (python:3.12)": {
        "base": 890,
        "build_tools": 120,
        "pip_cache": 85,
        "dependencies": 95,
        "source": 5,
    },
    "멀티 스테이지 (python:3.12-slim)": {
        "base": 130,
        "dependencies": 95,
        "source": 5,
    },
}

for stage_name, components in stages.items():
    total = sum(components.values())
    print(f"\n{stage_name}: ~{total}MB")
    for comp, size in components.items():
        bar = "█" * (size // 10)
        print(f"  {comp:20s} {size:4d}MB {bar}")

reduction = (1 - 230 / 1195) * 100
print(f"\n이미지 크기 절감: {reduction:.0f}%")
```

```output

단일 스테이지 (python:3.12): ~1195MB
  base                  890MB ████████████████████████████████████████████████████████████████████████████████████████
  build_tools           120MB ████████████
  pip_cache              85MB ████████
  dependencies           95MB █████████
  source                  5MB 

멀티 스테이지 (python:3.12-slim): ~230MB
  base                  130MB █████████████
  dependencies           95MB █████████
  source                  5MB 

이미지 크기 절감: 81%
```

### 개념 3: 환경 변수로 설정 주입

> 💡 **비유**: 환경 변수는 연극의 "무대 지시서"와 같습니다. 같은 대본(코드)이라도 무대 지시서(환경 변수)가 다르면 — 조명(포트), 소품(DB 연결), 의상(로그 레벨) — 전혀 다른 공연이 됩니다. 코드를 고치지 않고도 서버의 동작을 바꿀 수 있죠.

MCP 서버에서 하드코딩하면 안 되는 것들이 있습니다: 데이터베이스 URL, API 키, 포트 번호, 로그 레벨 등이죠. 이런 값들은 환경 변수로 외부에서 주입해야 합니다. 특히 [Ch11에서 설정한 OAuth 2.0 인증](11-ch11-인증과-보안/02-02-oauth-2.0-인증-통합.md)이나 [API 키·시크릿 관리](11-ch11-인증과-보안/03-03-api-키-관리와-시크릿-보호.md) 설정값은 코드에 절대 남겨두면 안 되는 대표적인 항목입니다.

```python
# config.py — 환경 변수 기반 설정 관리
import os
from dataclasses import dataclass, field


@dataclass
class ServerConfig:
    """MCP 서버 설정. 환경 변수에서 읽어옵니다."""

    # 서버 기본 설정
    server_name: str = field(
        default_factory=lambda: os.getenv("MCP_SERVER_NAME", "mcp-server")
    )
    host: str = field(
        default_factory=lambda: os.getenv("MCP_HOST", "0.0.0.0")
    )
    port: int = field(
        default_factory=lambda: int(os.getenv("MCP_PORT", "8000"))
    )

    # 데이터베이스
    database_url: str = field(
        default_factory=lambda: os.getenv("DATABASE_URL", "sqlite:///data.db")
    )

    # 로깅
    log_level: str = field(
        default_factory=lambda: os.getenv("MCP_LOG_LEVEL", "INFO")
    )

    # 보안 — 기본 인증
    auth_token: str = field(
        default_factory=lambda: os.getenv("MCP_AUTH_TOKEN", "")
    )

    # OAuth 2.0 인증 (Ch11 연동)
    oauth_client_id: str = field(
        default_factory=lambda: os.getenv("OAUTH_CLIENT_ID", "")
    )
    oauth_client_secret: str = field(
        default_factory=lambda: os.getenv("OAUTH_CLIENT_SECRET", "")
    )
    oauth_issuer: str = field(
        default_factory=lambda: os.getenv("OAUTH_ISSUER", "")
    )
    oauth_jwks_url: str = field(
        default_factory=lambda: os.getenv("OAUTH_JWKS_URL", "")
    )
    jwt_signing_key: str = field(
        default_factory=lambda: os.getenv("JWT_SIGNING_KEY", "")
    )

    @property
    def is_production(self) -> bool:
        return os.getenv("MCP_ENV", "development") == "production"

    @property
    def has_oauth(self) -> bool:
        """OAuth 설정이 완비되었는지 확인합니다."""
        return all([self.oauth_client_id, self.oauth_issuer, self.oauth_jwks_url])
```

환경 변수를 주입하는 방법은 여러 가지입니다:

```dockerfile
# Dockerfile에서 기본값 설정
ENV MCP_SERVER_NAME=my-server
ENV MCP_PORT=8000
ENV MCP_LOG_LEVEL=INFO
```

```bash
# docker run에서 개별 주입
docker run -d \
  -e MCP_PORT=9000 \
  -e DATABASE_URL=postgresql://user:pass@db:5432/mcp \
  -e MCP_LOG_LEVEL=DEBUG \
  mcp-server

# .env 파일로 일괄 주입
docker run -d --env-file .env mcp-server
```

```ini
# .env 파일 예시 (절대 Git에 커밋하지 마세요!)
MCP_SERVER_NAME=production-server
MCP_PORT=8000
DATABASE_URL=postgresql://admin:secret@db-host:5432/mcp_prod
MCP_AUTH_TOKEN=sk-prod-xxxxxxxxxxxx
MCP_LOG_LEVEL=WARNING
MCP_ENV=production

# OAuth 2.0 인증 설정 (Ch11에서 구성한 값들)
OAUTH_CLIENT_ID=mcp-server-prod-a1b2c3
OAUTH_CLIENT_SECRET=your-client-secret-here
OAUTH_ISSUER=https://auth.example.com
OAUTH_JWKS_URL=https://auth.example.com/.well-known/jwks.json
JWT_SIGNING_KEY=your-jwt-signing-key-here
```

[Ch11의 OAuth 2.0 통합](11-ch11-인증과-보안/02-02-oauth-2.0-인증-통합.md)에서 설정한 `client_id`, `client_secret`, JWKS URL, issuer 같은 값들을 Docker 환경에서 관리하는 방법을 좀 더 구체적으로 살펴보겠습니다. 이 설정값들은 인증 제공자(Auth0, Keycloak, 자체 인증 서버 등)에서 발급받은 것인데, 환경(개발/스테이징/프로덕션)마다 서로 다른 값을 사용해야 합니다.

> 📊 **그림 6**: Docker 환경에서의 OAuth 설정값 주입 흐름

```mermaid
flowchart LR
    subgraph AUTH["인증 제공자"]
        AP["Auth0 / Keycloak"]
    end

    subgraph INJECT["설정 주입"]
        ENV[".env 파일<br/>또는 Docker Secrets"]
        COMPOSE["docker-compose.yml<br/>environment 섹션"]
    end

    subgraph MCP["MCP 서버 컨테이너"]
        CFG["ServerConfig<br/>os.getenv()"]
        VERIFY["JWT 검증<br/>JWKS URL로 공개키 조회"]
    end

    AP -->|"client_id, secret,<br/>issuer, JWKS URL 발급"| ENV
    ENV --> COMPOSE
    COMPOSE -->|"환경 변수 주입"| CFG
    CFG --> VERIFY
    VERIFY -->|"토큰 검증 요청"| AP

    style AUTH fill:#9C27B0,color:#fff
    style INJECT fill:#FF9800,color:#fff
    style MCP fill:#4CAF50,color:#fff
```

Docker Compose에서 OAuth 설정을 주입하는 예시를 보겠습니다:

```yaml
# docker-compose.yml — OAuth 설정이 포함된 프로덕션 구성
services:
  mcp-server:
    build: .
    ports:
      - "8000:8000"
    environment:
      # 서버 기본 설정
      - MCP_SERVER_NAME=product-mcp
      - MCP_PORT=8000
      - DATABASE_URL=postgresql://mcp_user:mcp_pass@db:5432/products
      # OAuth 2.0 인증 — Ch11에서 설정한 값들
      - OAUTH_CLIENT_ID=${OAUTH_CLIENT_ID}        # .env에서 참조
      - OAUTH_CLIENT_SECRET=${OAUTH_CLIENT_SECRET}
      - OAUTH_ISSUER=${OAUTH_ISSUER}
      - OAUTH_JWKS_URL=${OAUTH_JWKS_URL}
      - JWT_SIGNING_KEY=${JWT_SIGNING_KEY}
    env_file:
      - .env  # 민감한 값은 .env에서 일괄 로드
```

`${VARIABLE}` 구문을 사용하면, `docker-compose.yml` 파일 자체에는 실제 시크릿이 노출되지 않으면서도 `.env` 파일의 값을 참조할 수 있습니다. 이렇게 하면 `docker-compose.yml`은 안심하고 Git에 커밋할 수 있죠.

프로덕션 환경에서 보안 수준을 한 단계 더 높이려면, Docker Secrets를 활용할 수도 있습니다:

```yaml
# docker-compose.yml — Docker Secrets로 민감 정보 보호
services:
  mcp-server:
    build: .
    environment:
      - OAUTH_CLIENT_ID_FILE=/run/secrets/oauth_client_id
      - OAUTH_CLIENT_SECRET_FILE=/run/secrets/oauth_client_secret
      - JWT_SIGNING_KEY_FILE=/run/secrets/jwt_signing_key
    secrets:
      - oauth_client_id
      - oauth_client_secret
      - jwt_signing_key

secrets:
  oauth_client_id:
    file: ./secrets/oauth_client_id.txt
  oauth_client_secret:
    file: ./secrets/oauth_client_secret.txt
  jwt_signing_key:
    file: ./secrets/jwt_signing_key.txt
```

Docker Secrets를 사용하면 시크릿이 컨테이너 내부의 `/run/secrets/` 디렉토리에 파일로 마운트됩니다. 환경 변수와 달리 `docker inspect`나 프로세스 목록에 값이 노출되지 않아 더 안전합니다. 서버 코드에서는 `_FILE` 접미사가 붙은 환경 변수를 감지하여 파일에서 값을 읽도록 처리합니다:

```python
# config.py — Docker Secrets 지원 헬퍼
def get_secret(env_name: str, default: str = "") -> str:
    """환경 변수 또는 Docker Secrets 파일에서 값을 읽습니다."""
    # 먼저 _FILE 변형 확인 (Docker Secrets)
    file_path = os.getenv(f"{env_name}_FILE")
    if file_path and os.path.isfile(file_path):
        with open(file_path) as f:
            return f.read().strip()
    # 일반 환경 변수 폴백
    return os.getenv(env_name, default)
```

> 🔥 **실무 팁**: `.env` 파일은 반드시 `.gitignore`에 추가하세요. 대신 `.env.example` 파일을 커밋하여 어떤 환경 변수가 필요한지 문서화합니다. `OAUTH_CLIENT_SECRET=your-secret-here`처럼 실제 값 대신 플레이스홀더를 넣어 두세요. OAuth 설정값은 환경별(dev/staging/prod)로 반드시 다른 값을 사용해야 합니다 — 개발 환경의 `client_secret`이 프로덕션에 노출되면 보안 사고로 이어질 수 있습니다.

### 개념 4: Docker Compose로 다중 서비스 구성

> 💡 **비유**: Docker Compose는 오케스트라 지휘자와 같습니다. 바이올린(MCP 서버), 첼로(데이터베이스), 플루트(리버스 프록시) — 각 악기가 독립적으로 연주하지만 지휘자가 동시에 시작시키고, 박자를 맞추고, 연습 환경을 세팅합니다. `docker compose up` 한 번이면 전체 오케스트라가 연주를 시작하죠.

실전에서 MCP 서버는 혼자 동작하지 않습니다. [Ch7에서 만든 데이터베이스 연동 서버](07-ch7-실전-서버-데이터베이스-연동/01-01-db-서버-설계-전략.md)처럼 DB가 필요하고, 프로덕션에서는 리버스 프록시도 필요합니다.

```yaml
# docker-compose.yml — MCP 서버 + PostgreSQL + Nginx
services:
  # MCP 서버 (Streamable HTTP)
  mcp-server:
    build:
      context: .
      dockerfile: Dockerfile.multistage
    ports:
      - "8000:8000"
    environment:
      - MCP_SERVER_NAME=ecommerce-mcp
      - MCP_PORT=8000
      - DATABASE_URL=postgresql://mcp_user:mcp_pass@db:5432/mcp_db
      - MCP_LOG_LEVEL=INFO
      - MCP_ENV=production
      - PYTHONUNBUFFERED=1
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    # 리소스 제한
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M

  # PostgreSQL 데이터베이스
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: mcp_user
      POSTGRES_PASSWORD: mcp_pass
      POSTGRES_DB: mcp_db
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U mcp_user -d mcp_db"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Nginx 리버스 프록시 (SSE 스트리밍 지원)
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - mcp-server

volumes:
  pgdata:
```

Nginx 설정에서 SSE 스트리밍을 위해 반드시 버퍼링을 비활성화해야 합니다:

```nginx
# nginx.conf — MCP Streamable HTTP용 설정
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://mcp-server:8000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Connection '';

        # SSE 스트리밍에 필수!
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;

        # chunked transfer encoding 지원
        chunked_transfer_encoding on;
    }
}
```

> 📊 **그림 4**: Docker Compose 멀티 서비스 아키텍처

```mermaid
flowchart TD
    CLIENT["MCP Client"] -->|"HTTP :80"| NGINX["Nginx<br/>리버스 프록시"]
    NGINX -->|"proxy_pass :8000"| MCP["MCP Server<br/>FastMCP + Streamable HTTP"]
    MCP -->|"postgresql:5432"| DB["PostgreSQL<br/>16-alpine"]
    DB --> VOL["pgdata<br/>영속 볼륨"]

    MCP -.->|"healthcheck"| DB
    NGINX -.->|"depends_on"| MCP

    style CLIENT fill:#9C27B0,color:#fff
    style NGINX fill:#2196F3,color:#fff
    style MCP fill:#4CAF50,color:#fff
    style DB fill:#FF9800,color:#fff
    style VOL fill:#607D8B,color:#fff
```

핵심 포인트 몇 가지를 짚어보겠습니다:

- **`depends_on` + `condition: service_healthy`**: DB가 완전히 준비된 후에만 MCP 서버가 시작됩니다. 단순 `depends_on`만으로는 부족해요 — 컨테이너가 "시작"된 것과 "준비"된 것은 다르거든요.
- **`deploy.resources.limits`**: CPU와 메모리 제한은 Docker MCP Catalog의 기본 정책(1 CPU, 2GB)을 참고한 것입니다.
- **`volumes`**: DB 데이터는 반드시 명명된 볼륨에 저장해야 합니다. 컨테이너가 재시작되어도 데이터가 유지됩니다.

### 개념 5: Docker MCP Toolkit과 Catalog 활용

> 💡 **비유**: Docker MCP Catalog는 "앱스토어"와 같습니다. npm이나 PyPI에서 패키지를 검색하고 설치하듯이, MCP Catalog에서 검증된 MCP 서버를 검색하고 한 줄 명령어로 실행할 수 있습니다. Docker MCP Toolkit은 이 앱스토어를 관리하는 "홈 화면"이고요.

Docker는 2025년부터 MCP 생태계에 적극적으로 뛰어들었습니다. 두 가지 핵심 제품을 알아야 합니다:

**Docker MCP Catalog:**
- Docker Hub의 `mcp/` 네임스페이스에 수백 개의 검증된 MCP 서버가 게시되어 있습니다 (2026년 3월 기준 270개 이상)
- Grafana, Neo4j, Pulumi, Heroku 등 주요 벤더가 공식 서버를 제공합니다
- 출시 수 주 만에 100만 회 이상 pull 되었습니다

**Docker MCP Toolkit:**
- Docker Desktop 4.59+ 에 내장된 무료 기능입니다
- MCP 서버를 프로필로 묶어 관리하고, OCI 레지스트리를 통해 팀과 공유할 수 있습니다
- Claude Desktop, Cursor, VS Code 등 주요 MCP 클라이언트와 바로 연결됩니다

Claude Desktop에서 Docker MCP 서버를 사용하는 설정은 이렇게 간단합니다:

```json
{
    "mcpServers": {
        "sqlite": {
            "command": "docker",
            "args": [
                "run", "-i", "--rm", "--init",
                "-e", "DOCKER_CONTAINER=true",
                "-v", "/path/to/data:/data",
                "mcp/sqlite",
                "--db-path", "/data/mydb.sqlite"
            ]
        }
    }
}
```

여러분이 만든 MCP 서버를 Catalog에 올리려면, Docker Hub에 `mcp/` 네임스페이스가 아닌 자신의 네임스페이스로 push하거나, Docker MCP Gateway를 통해 커스텀 Catalog를 구성할 수 있습니다:

```bash
# 자신의 MCP 서버를 Docker Hub에 게시
docker build -t yourusername/mcp-ecommerce:1.0 .
docker push yourusername/mcp-ecommerce:1.0

# Claude Desktop에서 사용
# claude_desktop_config.json의 mcpServers에 추가
```

### 개념 6: 보안 모범 사례

> 💡 **비유**: Docker 컨테이너의 보안은 은행 금고와 같습니다. 금고(컨테이너)가 있다고 해서 자동으로 안전한 게 아닙니다. 금고 벽 두께(최소 이미지), 잠금장치(non-root 사용자), 접근 권한(read-only 파일시스템), 감시 카메라(헬스체크)까지 갖춰야 진정한 보안이죠.

MCP 서버는 LLM이 직접 호출하는 도구를 제공하기 때문에, 보안이 특히 중요합니다. [Ch11에서 다룬 보안 위협과 방어 전략](11-ch11-인증과-보안/04-04-보안-위협과-방어-전략.md)을 Docker 레벨에서도 방어해야 합니다. 애플리케이션 수준의 인증·인가는 [OAuth 2.0 통합](11-ch11-인증과-보안/02-02-oauth-2.0-인증-통합.md)과 [API 키 관리](11-ch11-인증과-보안/03-03-api-키-관리와-시크릿-보호.md)를 참고하세요.

```dockerfile
# Dockerfile.secure — 보안 강화 버전
FROM python:3.12-slim AS builder

WORKDIR /build
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# -------------------------------------------
FROM python:3.12-slim

# 1. 최소 패키지만 유지 (불필요한 도구 제거)
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get autoremove -y && \
    rm -rf /var/lib/apt/lists/*

# 2. non-root 사용자 생성
RUN groupadd -r mcp && \
    useradd -r -g mcp -d /app -s /sbin/nologin mcp

# 3. 가상 환경 복사
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

WORKDIR /app

# 4. 소스 코드 복사 (소유자 지정)
COPY --chown=mcp:mcp server.py config.py ./

# 5. 환경 변수
ENV PYTHONUNBUFFERED=1
ENV MCP_ENV=production

# 6. non-root로 전환
USER mcp

# 7. 헬스체크
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

EXPOSE 8000

CMD ["python", "server.py"]
```

> 📊 **그림 5**: Docker 보안 계층

```mermaid
flowchart TD
    A["최소 베이스 이미지<br/>python:3.12-slim"] --> B["불필요한 패키지 제거<br/>apt autoremove"]
    B --> C["non-root 사용자<br/>USER mcp"]
    C --> D["리소스 제한<br/>CPU 1, Memory 512M"]
    D --> E["네트워크 격리<br/>필요한 포트만 노출"]
    E --> F["읽기 전용 파일시스템<br/>--read-only"]
    F --> G["시크릿 관리<br/>환경 변수 / Docker Secrets"]

    style A fill:#E3F2FD,color:#333
    style B fill:#BBDEFB,color:#333
    style C fill:#90CAF9,color:#333
    style D fill:#64B5F6,color:#fff
    style E fill:#42A5F5,color:#fff
    style F fill:#2196F3,color:#fff
    style G fill:#1565C0,color:#fff
```

보안을 더 강화하려면 실행 시 추가 플래그를 사용합니다:

```bash
# 프로덕션 실행 명령어 — 보안 강화 옵션
docker run -d \
  --name mcp-server \
  --read-only \                    # 파일시스템 읽기 전용
  --tmpfs /tmp:size=64m \          # tmp만 쓰기 허용
  --cap-drop ALL \                 # 모든 Linux capability 제거
  --security-opt no-new-privileges \  # 권한 상승 방지
  --memory 512m \                  # 메모리 제한
  --cpus 1.0 \                    # CPU 제한
  -p 8000:8000 \
  --env-file .env \
  mcp-server:latest
```

## 실습: 직접 해보기

실제로 MCP 서버를 Docker로 컨테이너화하고 Docker Compose로 DB와 함께 실행해 봅시다. [Ch7의 e-commerce 서버](07-ch7-실전-서버-데이터베이스-연동/05-05-e-commerce-데이터-서버-실습.md)를 간소화한 버전을 사용합니다.

**프로젝트 구조:**

```
mcp-docker-lab/
├── server.py          # MCP 서버 코드
├── config.py          # 환경 변수 설정
├── requirements.txt   # 의존성
├── Dockerfile         # 멀티 스테이지 빌드
├── docker-compose.yml # 서버 + DB 구성
├── nginx.conf         # 리버스 프록시 설정
├── init.sql           # DB 초기화 스크립트
├── .env.example       # 환경 변수 템플릿
└── .dockerignore      # Docker 빌드 제외 파일
```

**Step 1: 서버 코드 작성**

```python
# server.py — Docker 배포용 MCP 서버
import os
import logging
from mcp.server.fastmcp import FastMCP

# 환경 변수에서 설정 로드
SERVER_NAME = os.getenv("MCP_SERVER_NAME", "product-mcp")
LOG_LEVEL = os.getenv("MCP_LOG_LEVEL", "INFO")
DB_URL = os.getenv("DATABASE_URL", "sqlite:///products.db")

logging.basicConfig(level=getattr(logging, LOG_LEVEL))
logger = logging.getLogger(SERVER_NAME)

mcp = FastMCP(SERVER_NAME)


@mcp.tool()
async def search_products(query: str, limit: int = 10) -> str:
    """상품을 검색합니다. 키워드로 상품명과 설명을 검색합니다."""
    import asyncpg

    # DATABASE_URL에서 PostgreSQL 연결 정보 파싱
    conn = await asyncpg.connect(DB_URL.replace("postgresql://", "postgres://"))
    try:
        rows = await conn.fetch(
            """
            SELECT id, name, price, description
            FROM products
            WHERE name ILIKE $1 OR description ILIKE $1
            LIMIT $2
            """,
            f"%{query}%",
            limit,
        )
        if not rows:
            return f"'{query}'에 해당하는 상품이 없습니다."

        results = []
        for row in rows:
            results.append(
                f"[{row['id']}] {row['name']} - ₩{row['price']:,.0f}\n"
                f"  {row['description']}"
            )
        return f"검색 결과 ({len(rows)}건):\n" + "\n".join(results)
    finally:
        await conn.close()


@mcp.tool()
async def get_product_stats() -> str:
    """전체 상품 통계를 반환합니다."""
    import asyncpg

    conn = await asyncpg.connect(DB_URL.replace("postgresql://", "postgres://"))
    try:
        stats = await conn.fetchrow(
            """
            SELECT
                COUNT(*) as total,
                AVG(price) as avg_price,
                MIN(price) as min_price,
                MAX(price) as max_price
            FROM products
            """
        )
        return (
            f"상품 통계:\n"
            f"  총 상품 수: {stats['total']}\n"
            f"  평균 가격: ₩{stats['avg_price']:,.0f}\n"
            f"  최저 가격: ₩{stats['min_price']:,.0f}\n"
            f"  최고 가격: ₩{stats['max_price']:,.0f}"
        )
    finally:
        await conn.close()


# 헬스체크 엔드포인트 (Streamable HTTP용)
@mcp.custom_route("/health", methods=["GET"])
async def health_check(request):
    from starlette.responses import JSONResponse

    return JSONResponse({"status": "healthy", "service": SERVER_NAME})


if __name__ == "__main__":
    transport = os.getenv("MCP_TRANSPORT", "streamable-http")
    host = os.getenv("MCP_HOST", "0.0.0.0")
    port = int(os.getenv("MCP_PORT", "8000"))

    logger.info(f"Starting {SERVER_NAME} on {host}:{port} ({transport})")
    mcp.run(transport=transport, host=host, port=port)
```

**Step 2: 의존성과 Dockerfile**

```
# requirements.txt
mcp[cli]>=1.25,<2
asyncpg>=0.30.0
```

```dockerfile
# Dockerfile
# Stage 1: Builder
FROM python:3.12-slim AS builder
WORKDIR /build
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Stage 2: Runtime
FROM python:3.12-slim
RUN groupadd -r mcp && useradd -r -g mcp -d /app -s /sbin/nologin mcp
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
ENV PYTHONUNBUFFERED=1
WORKDIR /app
COPY --chown=mcp:mcp server.py .
USER mcp
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1
CMD ["python", "server.py"]
```

**Step 3: Docker Compose 설정**

```yaml
# docker-compose.yml
services:
  mcp-server:
    build: .
    ports:
      - "8000:8000"
    environment:
      - MCP_SERVER_NAME=product-mcp
      - MCP_TRANSPORT=streamable-http
      - MCP_PORT=8000
      - DATABASE_URL=postgresql://mcp_user:mcp_pass@db:5432/products
      - MCP_LOG_LEVEL=INFO
      - PYTHONUNBUFFERED=1
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: mcp_user
      POSTGRES_PASSWORD: mcp_pass
      POSTGRES_DB: products
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U mcp_user -d products"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

```sql
-- init.sql — 데이터베이스 초기화
CREATE TABLE IF NOT EXISTS products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    price NUMERIC(10, 2) NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO products (name, price, description) VALUES
    ('Python 입문서', 35000, 'Python 기초부터 고급까지 다루는 완전 가이드'),
    ('MCP 실전 가이드', 42000, 'Model Context Protocol의 모든 것'),
    ('Docker 마스터', 38000, '컨테이너 기술의 기초부터 오케스트레이션까지'),
    ('AI 에이전트 설계', 55000, 'LLM 기반 에이전트 아키텍처와 구현'),
    ('FastAPI 웹 개발', 32000, '현대적 Python 웹 프레임워크 완벽 가이드');
```

**Step 4: 빌드 및 실행**

```bash
# 전체 서비스 빌드 및 실행
docker compose up --build -d

# 로그 확인
docker compose logs -f mcp-server

# 헬스체크 확인
curl http://localhost:8000/health

# 서비스 중지
docker compose down

# 데이터 볼륨 포함 완전 정리
docker compose down -v
```

**Step 5: .dockerignore 작성**

```
# .dockerignore — 빌드 컨텍스트에서 제외
.git
.gitignore
.env
*.pyc
__pycache__
.venv
venv
*.egg-info
.mypy_cache
.pytest_cache
docker-compose.yml
README.md
```

## 더 깊이 알아보기

### Docker와 MCP의 만남 — Solomon Hykes의 비전에서 AI 인프라까지

Docker의 탄생 배경을 알면 MCP와의 결합이 왜 자연스러운지 이해할 수 있습니다.

2013년, Solomon Hykes가 PyCon에서 Docker를 처음 공개했을 때, 핵심 메시지는 "Build once, run anywhere"였습니다. 화물 컨테이너가 규격화되면서 전 세계 물류 혁명이 일어난 것처럼, 소프트웨어도 표준 컨테이너에 담으면 어디서든 동일하게 실행할 수 있다는 아이디어였죠.

10년이 지난 2024년 12월, Docker는 Anthropic과 협력하여 MCP 서버의 표준 배포 방식으로 Docker를 채택했습니다. David Soria Parra(Anthropic)와 Jim Clark(Docker)이 함께 발표한 블로그 포스트에서, MCP 서버의 Docker 컨테이너화가 풀어야 할 네 가지 문제를 정의했습니다: 환경 일관성, 의존성 격리, 크로스 플랫폼 지원, 보안 샌드박싱. 놀랍게도, 이 네 가지는 2013년 Hykes가 Docker를 소개할 때 제시한 문제와 정확히 같았습니다.

2025년에 Docker MCP Catalog이 출시되면서, `mcp/` 네임스페이스가 Docker Hub에 등장했습니다. 수백 개의 검증된 MCP 서버가 게시되었고, 수 주 만에 누적 100만 회 이상 pull 되었습니다. 이는 MCP 생태계가 "직접 빌드"에서 "검증된 컨테이너 사용"으로 전환되고 있음을 보여주는 수치입니다.

### 멀티 스테이지 빌드의 기원

멀티 스테이지 빌드는 Docker 17.05(2017년 5월)에 도입되었습니다. 그 전에는 "빌더 패턴(builder pattern)"이라 불리는 기법 — 빌드용 Dockerfile과 실행용 Dockerfile을 따로 만들고 쉘 스크립트로 연결하는 — 을 사용했는데, 복잡하고 오류가 잦았죠. 멀티 스테이지 빌드는 이 과정을 하나의 Dockerfile 안에 우아하게 통합한 것입니다.

## 흔한 오해와 팁

> ⚠️ **흔한 오해**: "Docker 컨테이너 안에서는 보안 걱정을 안 해도 된다"는 생각은 위험합니다. 컨테이너는 VM(가상 머신)이 아닙니다. 기본 설정으로는 호스트 커널을 공유하고, root로 실행되며, 파일시스템에 쓰기 권한이 있습니다. non-root 사용자, `--read-only`, `--cap-drop ALL` 같은 추가 조치가 반드시 필요합니다.

> 💡 **알고 계셨나요?**: Docker MCP Catalog의 각 서버 컨테이너는 기본적으로 CPU 1코어, 메모리 2GB로 제한됩니다. 이 "관대하지만 안전한" 제한은 하나의 MCP 서버가 호스트 전체 리소스를 독점하는 것을 방지하면서도, 대부분의 MCP 도구 실행에는 충분한 리소스를 보장하기 위한 것입니다.

> 🔥 **실무 팁**: `PYTHONUNBUFFERED=1`을 잊으면 stdio Transport MCP 서버가 "아무 반응 없이 멈추는" 현상이 발생합니다. 디버깅하기 매우 어려운 문제인데, 원인은 단순히 Python stdout 버퍼링입니다. `Dockerfile`에 ENV로 고정하고, `docker-compose.yml`에서도 명시적으로 선언하세요. 보험은 이중으로 드는 게 좋습니다.

## 핵심 정리

| 개념 | 설명 |
|------|------|
| `PYTHONUNBUFFERED=1` | MCP stdio Transport에 필수. Python stdout 버퍼링을 비활성화하여 JSON-RPC 메시지가 즉시 전달되도록 함 |
| 멀티 스테이지 빌드 | Builder 스테이지에서 의존성 설치, Runtime 스테이지에서 venv만 복사. 이미지 크기 70-80% 절감 |
| `docker run -i` | stdio Transport용. stdin을 열어 MCP 클라이언트와 양방향 통신 |
| `docker run -d -p` | Streamable HTTP Transport용. 백그라운드 실행 + 포트 매핑 |
| Docker Compose | MCP 서버 + DB + 프록시를 하나의 설정 파일로 관리. `depends_on` + `healthcheck`로 기동 순서 보장 |
| non-root 사용자 | `RUN useradd -r mcp` + `USER mcp`로 보안 강화. 컨테이너 탈출 시 피해 최소화 |
| Docker MCP Catalog | Docker Hub `mcp/` 네임스페이스에 수백 개의 검증된 MCP 서버. `docker run` 한 줄로 사용 |
| 환경 변수 주입 | `-e`, `--env-file`, `docker-compose.yml`의 `environment`로 설정 분리. OAuth 시크릿은 Docker Secrets 권장 |
| SSE 프록시 설정 | Nginx `proxy_buffering off` 필수. 버퍼링이 SSE 스트리밍을 차단함 |
| 헬스체크 | `HEALTHCHECK` 지시자 또는 Compose `healthcheck`로 서비스 상태 모니터링 |

## 다음 섹션 미리보기

Docker로 컨테이너를 만들었으니, 이제 이 컨테이너를 실제 서버에 올려야겠죠? 다음 섹션 [Streamable HTTP 서버 배포](13-ch13-배포와-운영/02-02-streamable-http-서버-배포.md)에서는 Streamable HTTP Transport를 사용한 MCP 서버를 ASGI 서버(Uvicorn/Gunicorn)에 올리고, TLS 인증서를 설정하고, 프로덕션 수준의 HTTP 서버로 완성하는 방법을 다룹니다.

## 참고 자료

- [The Model Context Protocol: Simplifying Building AI Apps with Anthropic Claude Desktop and Docker](https://www.docker.com/blog/the-model-context-protocol-simplifying-building-ai-apps-with-anthropic-claude-desktop-and-docker/) - Docker + Anthropic 공동 발표. MCP 서버 Docker화의 동기와 아키텍처 설명
- [How to Build and Deploy a Model Context Protocol (MCP) Server](https://northflank.com/blog/how-to-build-and-deploy-a-model-context-protocol-mcp-server) - Northflank 플랫폼에서 MCP 서버 빌드부터 배포까지의 실전 가이드
- [Docker MCP Catalog and Toolkit Documentation](https://docs.docker.com/ai/mcp-catalog-and-toolkit/) - Docker MCP Catalog/Toolkit 공식 문서. 서버 검색, 프로필 관리, 클라이언트 연결 방법
- [Build to Prod: MCP Servers with Docker](https://www.docker.com/blog/build-to-prod-mcp-servers-with-docker/) - Docker 공식 블로그의 MCP 서버 프로덕션 배포 가이드
- [FastMCP HTTP Deployment Guide](https://gofastmcp.com/deployment/http) - FastMCP의 HTTP Transport 배포 설정, stateless 모드, ASGI 통합 가이드
- [MCP Python SDK (GitHub)](https://github.com/modelcontextprotocol/python-sdk) - MCP Python SDK 공식 저장소. FastMCP, Transport 구현, 예제 코드

---
### 🔗 Related Sessions
- [stdio transport](02-ch2-mcp-아키텍처와-프로토콜-구조/02-02-transport-계층-stdio.md) (prerequisite)
- [streamable http transport](01-ch1-mcp의-탄생과-설계-철학/04-04-mcp-스펙-변천사와-2026-로드맵.md) (prerequisite)
