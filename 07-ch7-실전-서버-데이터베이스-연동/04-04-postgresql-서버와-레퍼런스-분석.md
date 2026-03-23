# PostgreSQL 서버와 레퍼런스 분석

> 공식 레퍼런스 서버의 설계를 해부하고, asyncpg 커넥션 풀 기반 Python PostgreSQL MCP 서버를 구축합니다.

## 개요

이 섹션에서는 Anthropic이 공개한 공식 PostgreSQL MCP 레퍼런스 서버의 코드를 분석하고, 그 설계 패턴을 Python으로 재구현합니다. SQLite에서 PostgreSQL로 넘어가면서 달라지는 점—커넥션 풀링, `$1` 파라미터 바인딩, 트랜잭션 격리 수준, 스키마 인트로스펙션—을 깊이 있게 다룹니다.

**선수 지식**: [SQLite MCP 서버 구축](07-ch7-실전-서버-데이터베이스-연동/02-02-sqlite-mcp-서버-구축.md)의 lifespan 패턴과 AppContext, [쿼리 안전성과 권한 제어](07-ch7-실전-서버-데이터베이스-연동/03-03-쿼리-안전성과-권한-제어.md)의 SecureQueryExecutor 개념

**학습 목표**:
- 공식 PostgreSQL MCP 레퍼런스 서버의 아키텍처와 설계 결정을 분석할 수 있다
- asyncpg 커넥션 풀을 FastMCP lifespan에 통합할 수 있다
- information_schema 기반 스키마 인트로스펙션 도구를 구현할 수 있다
- 다중 데이터베이스를 지원하는 MCP 서버를 설계할 수 있다

## 왜 알아야 할까?

앞선 세션에서 SQLite로 MCP DB 서버의 핵심 패턴을 익혔습니다. 하지만 실무에서 마주치는 데이터베이스는 대부분 PostgreSQL이죠. 2024년 Stack Overflow 조사에서 PostgreSQL은 전문 개발자가 가장 많이 사용하는 데이터베이스 1위를 차지했고, MCP 생태계에서도 PostgreSQL 서버는 가장 인기 있는 레퍼런스 서버 중 하나입니다.

SQLite에서 PostgreSQL로 넘어가면 단순히 드라이버만 바꾸면 될까요? 아닙니다. 커넥션 풀링, 네트워크 레이턴시, 동시 접속 관리, 트랜잭션 격리 수준 같은 **운영 수준의 고려사항**이 추가됩니다. Anthropic이 공개한 레퍼런스 서버는 이런 문제를 최소한의 코드로 어떻게 해결했는지 보여주는 훌륭한 교본입니다. 이 서버를 해부한 뒤, Python asyncpg로 더 강력한 버전을 직접 만들어 보겠습니다.

## 핵심 개념

### 개념 1: 공식 레퍼런스 서버 아키텍처 분석

> 💡 **비유**: 레퍼런스 서버는 "모범 답안"과 같습니다. 시험장에서 모범 답안을 그대로 베끼면 안 되지만, 왜 그 풀이법을 선택했는지 이해하면 응용력이 생기죠. 공식 서버의 핵심 코드에는 Anthropic 팀이 내린 핵심 설계 결정이 압축되어 있습니다.

Anthropic의 공식 PostgreSQL MCP 서버(`@modelcontextprotocol/server-postgres`)는 TypeScript로 작성된 레퍼런스 구현입니다. 초기 버전 기준 약 130줄의 간결한 코드로, MCP DB 서버의 정수를 담고 있죠. (이후 커뮤니티 기여와 기능 추가로 코드 규모는 변동될 수 있습니다.)

> 📊 **그림 1**: 공식 레퍼런스 서버의 구조

```mermaid
flowchart TD
    CLI["CLI 인자<br/>postgresql://..."] --> Pool["pg.Pool<br/>커넥션 풀"]
    Pool --> R["Resources"]
    Pool --> T["Tools"]
    
    R --> R1["list_resources<br/>public 스키마 테이블 목록"]
    R --> R2["read_resource<br/>postgres://host/table/schema"]
    
    T --> T1["query Tool<br/>읽기 전용 SQL 실행"]
    
    T1 --> TX["BEGIN TRANSACTION<br/>READ ONLY"]
    TX --> Exec["쿼리 실행"]
    Exec --> RB["ROLLBACK<br/>항상 롤백"]
    
    style TX fill:#FF9800,color:#fff
    style RB fill:#F44336,color:#fff
```

핵심 설계 결정을 하나씩 살펴보겠습니다.

**1. 단일 도구 원칙**: 서버가 노출하는 도구는 `query` 딱 하나입니다. [DB 서버 설계 전략](07-ch7-실전-서버-데이터베이스-연동/01-01-db-서버-설계-전략.md)에서 논의한 "읽기/쓰기 분리" 대신, **읽기 전용을 트랜잭션 레벨에서 강제**하는 접근을 택했습니다.

```python
# 레퍼런스 서버의 핵심 패턴을 Python으로 표현
async def execute_read_only_query(pool, sql: str) -> list[dict]:
    """BEGIN READ ONLY + ROLLBACK으로 읽기 전용 보장"""
    async with pool.acquire() as conn:
        # 트랜잭션을 읽기 전용으로 시작
        async with conn.transaction(readonly=True):
            rows = await conn.fetch(sql)
            return [dict(row) for row in rows]
        # transaction 블록을 벗어나면 자동 ROLLBACK
```

**2. 스키마를 Resource로 노출**: `information_schema.tables`에서 public 스키마의 테이블 목록을 가져오고, 각 테이블의 컬럼 정보를 `information_schema.columns`에서 읽어 Resource로 제공합니다. URI 형식은 `postgres://<host>/<table>/schema`입니다.

**3. 비밀번호 제거**: Resource URI를 생성할 때 연결 문자열에서 패스워드를 제거합니다. LLM이 비밀번호를 학습하거나 출력에 포함시키는 것을 방지하기 위한 안전 장치죠.

> ⚠️ **흔한 오해**: "공식 서버니까 그대로 프로덕션에 써도 되겠지?"라고 생각하기 쉽습니다. 하지만 레퍼런스 서버는 **학습용 최소 구현**입니다. SQL 인젝션 방어, 행 수 제한, 감사 로깅 같은 [7.3에서 배운 보안 계층](07-ch7-실전-서버-데이터베이스-연동/03-03-쿼리-안전성과-권한-제어.md)이 빠져 있습니다. 실제로 2025년 Datadog 연구팀이 이 서버의 SQL 인젝션 가능성을 지적한 바 있습니다.

### 개념 2: asyncpg 커넥션 풀링

> 💡 **비유**: 커넥션 풀은 **카풀 서비스**와 비슷합니다. 매번 새 차를 렌트하는 대신(커넥션 생성), 풀에 대기 중인 차를 빌려 쓰고 반납합니다. 차를 미리 여러 대 준비해 두면(min_size), 갑자기 수요가 몰려도(동시 MCP 요청) 대기 시간이 줄어들죠.

[SQLite 서버](07-ch7-실전-서버-데이터베이스-연동/02-02-sqlite-mcp-서버-구축.md)에서는 `aiosqlite.connect()`로 단일 커넥션을 유지했습니다. SQLite는 파일 기반 임베디드 DB라 네트워크 오버헤드가 없기 때문이죠. PostgreSQL은 네트워크 너머에 있는 서버이므로, 커넥션 생성에 TCP 핸드셰이크 + 인증이 필요합니다. 매 쿼리마다 이 과정을 반복하면 수십 밀리초의 레이턴시가 추가됩니다.

> 📊 **그림 2**: 커넥션 풀 vs 매번 새 커넥션

```mermaid
sequenceDiagram
    participant MCP as MCP Tool
    participant Pool as 커넥션 풀
    participant PG as PostgreSQL

    Note over MCP,PG: 커넥션 풀 사용 시
    MCP->>Pool: acquire()
    Pool-->>MCP: 기존 커넥션 반환 (0.1ms)
    MCP->>PG: SELECT ... ($1)
    PG-->>MCP: 결과
    MCP->>Pool: release()

    Note over MCP,PG: 매번 새 커넥션 생성 시
    MCP->>PG: TCP 핸드셰이크 (5ms)
    PG-->>MCP: 연결 수립
    MCP->>PG: 인증 (3ms)
    PG-->>MCP: 인증 완료
    MCP->>PG: SELECT ...
    PG-->>MCP: 결과
    MCP->>PG: 커넥션 종료
```

asyncpg는 Python에서 가장 빠른 PostgreSQL 드라이버입니다. C 확장으로 PostgreSQL 바이너리 프로토콜을 직접 구현해서 psycopg3 대비 약 5배 빠른 성능을 보여줍니다.

```python
import asyncpg

# 커넥션 풀 생성 — MCP 서버 시작 시 한 번만
pool = await asyncpg.create_pool(
    dsn="postgresql://user:pass@localhost:5432/mydb",
    min_size=2,          # 최소 유지 커넥션 (유휴 시에도)
    max_size=10,         # 최대 커넥션 (동시 요청 한도)
    max_queries=50000,   # 이 횟수 이후 커넥션 교체
    max_inactive_connection_lifetime=300.0,  # 5분 유휴 시 해제
    command_timeout=30,  # 쿼리 타임아웃 30초
)
```

asyncpg의 파라미터 바인딩은 `$1, $2, ...` 형식입니다. SQLite의 `?` 와 달리 번호를 명시하죠.

```run:python
# asyncpg 파라미터 바인딩 스타일 비교
sqlite_style = "SELECT * FROM users WHERE age > ? AND name = ?"
asyncpg_style = "SELECT * FROM users WHERE age > $1 AND name = $2"

print(f"SQLite:  {sqlite_style}")
print(f"asyncpg: {asyncpg_style}")
print(f"\nasyncpg는 번호 기반이라 같은 파라미터를 재사용할 수 있습니다:")
print(f"  SELECT * FROM t WHERE col1 = $1 OR col2 = $1  (같은 값 두 번)")
```

```output
SQLite:  SELECT * FROM users WHERE age > ? AND name = ?
asyncpg: SELECT * FROM users WHERE age > $1 AND name = $2

asyncpg는 번호 기반이라 같은 파라미터를 재사용할 수 있습니다:
  SELECT * FROM t WHERE col1 = $1 OR col2 = $1  (같은 값 두 번)
```

### 개념 3: 스키마 인트로스펙션 도구

> 💡 **비유**: 스키마 인트로스펙션은 **건물 안내도**입니다. 처음 방문한 건물에서 "3층에 회의실이 몇 개 있고, 각각 몇 명 수용 가능한지" 알려주는 것처럼, LLM에게 "이 데이터베이스에 어떤 테이블이 있고, 각 컬럼의 타입은 무엇인지" 알려줍니다. 안내도 없이는 길을 잃듯, 스키마 정보 없이는 LLM이 올바른 쿼리를 작성할 수 없습니다.

PostgreSQL의 `information_schema`는 SQL 표준에 정의된 메타데이터 뷰 모음입니다. 어떤 PostgreSQL 버전이든 동일한 인터페이스로 스키마를 탐색할 수 있죠.

> 📊 **그림 3**: 스키마 인트로스펙션 계층

```mermaid
flowchart LR
    IS["information_schema"] --> T["tables<br/>테이블 목록"]
    IS --> C["columns<br/>컬럼 정보"]
    IS --> TC["table_constraints<br/>제약 조건"]
    IS --> KCU["key_column_usage<br/>키 컬럼"]
    
    T --> R1["Resource<br/>schema://tables"]
    C --> R2["Resource<br/>schema://tables/users"]
    TC --> R3["Resource<br/>schema://constraints"]
    KCU --> R4["Resource<br/>schema://relations"]
    
    R1 --> LLM["LLM이 쿼리 작성에 활용"]
    R2 --> LLM
    R3 --> LLM
    R4 --> LLM
```

MCP 서버에서 스키마를 Resource로 노출하면, LLM은 도구를 호출하기 전에 먼저 데이터 구조를 파악할 수 있습니다. 이 순서가 중요한데요—[Resource 프리미티브](05-ch5-resources-데이터-노출-프리미티브/01-01-resource-프리미티브-이해.md)에서 배운 것처럼, Resource는 application-controlled이라 클라이언트가 자동으로 LLM 컨텍스트에 주입할 수 있기 때문입니다.

```python
async def get_table_schema(pool: asyncpg.Pool, table_name: str) -> str:
    """특정 테이블의 컬럼 정보를 마크다운으로 반환"""
    query = """
        SELECT column_name, data_type, is_nullable, column_default
        FROM information_schema.columns
        WHERE table_schema = 'public' AND table_name = $1
        ORDER BY ordinal_position
    """
    rows = await pool.fetch(query, table_name)
    
    # LLM이 읽기 쉬운 마크다운 테이블로 포맷팅
    lines = [f"## {table_name}", ""]
    lines.append("| 컬럼 | 타입 | NULL 허용 | 기본값 |")
    lines.append("|------|------|-----------|--------|")
    for r in rows:
        nullable = "YES" if r["is_nullable"] == "YES" else "NO"
        default = r["column_default"] or "-"
        lines.append(f"| {r['column_name']} | {r['data_type']} | {nullable} | {default} |")
    
    return "\n".join(lines)
```

> 🔥 **실무 팁**: 스키마 정보를 JSON 대신 마크다운 테이블로 반환하면 토큰 사용량이 30~40% 줄어듭니다. pgEdge 팀의 연구에 따르면, TSV(탭 구분) 형식이 가장 토큰 효율적이지만, LLM의 이해도와 토큰 효율 사이의 최적점은 마크다운 테이블입니다.

### 개념 4: 트랜잭션 격리와 읽기 전용 모드

공식 레퍼런스 서버가 `BEGIN TRANSACTION READ ONLY` + `ROLLBACK`을 사용하는 이유는 무엇일까요? PostgreSQL의 트랜잭션 격리 수준을 이해하면 답이 보입니다.

> 📊 **그림 4**: PostgreSQL 트랜잭션 격리 수준

```mermaid
flowchart TD
    RU["Read Uncommitted<br/>PostgreSQL에서는 Read Committed와 동일"] 
    RC["Read Committed<br/>기본값, 커밋된 데이터만 읽음"]
    RR["Repeatable Read<br/>트랜잭션 시작 시점 스냅샷"]
    S["Serializable<br/>완전 직렬화, 가장 엄격"]
    
    RU --> RC --> RR --> S
    
    MCP["MCP 서버 권장"] -.->|"읽기 전용 + Repeatable Read"| RR
    
    style RR fill:#4CAF50,color:#fff
    style MCP fill:#2196F3,color:#fff
```

asyncpg에서는 `conn.transaction()` 컨텍스트 매니저로 격리 수준을 제어합니다.

```python
async with pool.acquire() as conn:
    # 읽기 전용 + Repeatable Read
    # → 트랜잭션 동안 일관된 스냅샷 보장
    async with conn.transaction(
        readonly=True,
        isolation="repeatable_read"
    ):
        # 이 블록 안에서는 INSERT/UPDATE/DELETE가 에러 발생
        rows = await conn.fetch("SELECT * FROM orders WHERE total > $1", 1000)
        stats = await conn.fetchrow("SELECT COUNT(*), AVG(total) FROM orders")
        # → 두 쿼리가 동일한 시점의 데이터를 봄 (일관성)
    # 블록 종료 시 자동 ROLLBACK (readonly이므로 COMMIT 불필요)
```

왜 `readonly=True`만으로 충분하지 않을까요? MCP 도구 하나의 호출 안에서 여러 쿼리를 실행하는 경우가 있습니다. 예를 들어 "주문 목록 조회 + 통계 계산"을 한 번의 tool call로 처리할 때, Repeatable Read를 걸지 않으면 두 쿼리 사이에 다른 트랜잭션이 데이터를 변경할 수 있습니다. LLM에게 "100건의 주문이 있는데 합계는 99건 기준입니다"라는 모순된 정보를 줄 수 있죠.

### 개념 5: 다중 데이터베이스 지원

> 💡 **비유**: 하나의 MCP 서버로 여러 데이터베이스를 다루는 것은 **통합 리모컨**과 같습니다. TV, 에어컨, 오디오를 각각의 리모컨 대신 하나로 제어하듯, LLM이 하나의 MCP 서버를 통해 여러 데이터베이스에 접근합니다.

실무에서는 하나의 서비스가 여러 데이터베이스를 사용하는 경우가 흔합니다. 사용자 DB, 주문 DB, 로그 DB가 따로 있을 수 있죠. pg-mcp-server 같은 커뮤니티 프로젝트에서는 **커넥션 ID 패턴**으로 이 문제를 해결합니다.

> 📊 **그림 5**: 다중 데이터베이스 아키텍처

```mermaid
flowchart TD
    LLM["LLM"] --> Host["MCP Host"]
    Host --> Client["MCP Client"]
    Client --> Server["PostgreSQL MCP Server"]
    
    Server --> PM["PoolManager<br/>커넥션 풀 관리자"]
    
    PM --> P1["Pool: users-db<br/>min=2, max=5"]
    PM --> P2["Pool: orders-db<br/>min=2, max=10"]
    PM --> P3["Pool: analytics-db<br/>min=1, max=3"]
    
    P1 --> DB1["PostgreSQL<br/>사용자 DB"]
    P2 --> DB2["PostgreSQL<br/>주문 DB"]
    P3 --> DB3["PostgreSQL<br/>분석 DB"]
    
    style PM fill:#FF9800,color:#fff
```

아래의 `PoolManager`는 asyncpg의 `Pool`을 감싸는 커스텀 매니저 클래스입니다(urllib3의 `PoolManager`와는 무관합니다). 여러 데이터베이스의 커넥션 풀을 `db_id` 키로 등록하고 조회하는 역할을 합니다.

```python
from dataclasses import dataclass, field

@dataclass
class PoolManager:
    """다중 데이터베이스 커넥션 풀 관리자
    
    asyncpg.Pool을 감싸는 커스텀 클래스입니다.
    urllib3의 PoolManager와는 이름만 같을 뿐 관련이 없습니다.
    """
    _pools: dict[str, asyncpg.Pool] = field(default_factory=dict)
    
    async def register(
        self, 
        db_id: str, 
        dsn: str,
        min_size: int = 2,
        max_size: int = 10
    ) -> None:
        """새 데이터베이스 커넥션 풀 등록"""
        if db_id in self._pools:
            raise ValueError(f"Database '{db_id}' already registered")
        self._pools[db_id] = await asyncpg.create_pool(
            dsn=dsn, min_size=min_size, max_size=max_size,
            command_timeout=30
        )
    
    def get_pool(self, db_id: str) -> asyncpg.Pool:
        """등록된 풀 반환"""
        if db_id not in self._pools:
            raise KeyError(f"Unknown database: '{db_id}'. "
                          f"Available: {list(self._pools.keys())}")
        return self._pools[db_id]
    
    async def close_all(self) -> None:
        """모든 풀 정리"""
        for pool in self._pools.values():
            await pool.close()
        self._pools.clear()
```

도구 호출 시 `db_id` 파라미터로 대상 데이터베이스를 지정합니다. LLM은 먼저 "어떤 데이터베이스가 있는지" Resource를 확인한 뒤, 적절한 데이터베이스를 선택해서 쿼리를 실행합니다.

## 실습: 직접 해보기

이제 공식 레퍼런스 서버의 설계를 바탕으로, asyncpg 커넥션 풀 기반의 완전한 PostgreSQL MCP 서버를 구축해 보겠습니다.

### 프로젝트 구조

```
pg-mcp-server/
├── server.py          # MCP 서버 진입점
├── db.py              # 커넥션 풀 관리
├── schema_tools.py    # 스키마 인트로스펙션
├── query_tools.py     # 쿼리 실행 도구
├── .env               # 환경 변수
└── pyproject.toml
```

### Step 1: 의존성과 환경 설정

```python
# pyproject.toml
# [project]
# dependencies = ["mcp[cli]>=1.20", "asyncpg>=0.30", "python-dotenv>=1.0"]
```

```python
# .env
# DATABASE_URL=postgresql://user:password@localhost:5432/mydb
# ALLOWED_SCHEMAS=public
# MAX_ROWS=500
# READ_ONLY=true
```

### Step 2: 커넥션 풀 모듈 (db.py)

```python
# db.py — 커넥션 풀 싱글턴 관리
import os
import asyncpg
from dotenv import load_dotenv

load_dotenv()

_pool: asyncpg.Pool | None = None

async def get_pool() -> asyncpg.Pool:
    """커넥션 풀 싱글턴 반환 (없으면 생성)"""
    global _pool
    if _pool is None:
        _pool = await asyncpg.create_pool(
            dsn=os.getenv("DATABASE_URL"),
            min_size=2,           # 최소 2개 유지
            max_size=10,          # 최대 10개까지 확장
            max_queries=50000,    # 5만 쿼리마다 커넥션 교체
            max_inactive_connection_lifetime=300.0,  # 5분 유휴 → 해제
            command_timeout=30,   # 쿼리 30초 타임아웃
        )
    return _pool

async def close_pool() -> None:
    """서버 종료 시 풀 정리"""
    global _pool
    if _pool is not None:
        await _pool.close()
        _pool = None
```

### Step 3: 스키마 인트로스펙션 (schema_tools.py)

```python
# schema_tools.py — information_schema 기반 스키마 탐색
import asyncpg
import json

ALLOWED_SCHEMAS = {"public"}  # 환경 변수에서 로드 가능

async def list_tables(pool: asyncpg.Pool, schema: str = "public") -> list[dict]:
    """스키마 내 모든 테이블과 행 수 추정치"""
    if schema not in ALLOWED_SCHEMAS:
        raise ValueError(f"Schema '{schema}' is not allowed")
    
    query = """
        SELECT t.table_name,
               t.table_type,
               pg_stat.n_live_tup AS estimated_rows
        FROM information_schema.tables t
        LEFT JOIN pg_stat_user_tables pg_stat
            ON t.table_name = pg_stat.relname
        WHERE t.table_schema = $1
          AND t.table_type IN ('BASE TABLE', 'VIEW')
        ORDER BY t.table_name
    """
    rows = await pool.fetch(query, schema)
    return [dict(r) for r in rows]

async def get_columns(pool: asyncpg.Pool, table_name: str) -> list[dict]:
    """테이블의 컬럼 상세 정보"""
    query = """
        SELECT c.column_name, c.data_type, c.is_nullable,
               c.column_default, c.character_maximum_length
        FROM information_schema.columns c
        WHERE c.table_schema = 'public' AND c.table_name = $1
        ORDER BY c.ordinal_position
    """
    return [dict(r) for r in await pool.fetch(query, table_name)]

async def get_foreign_keys(pool: asyncpg.Pool, table_name: str) -> list[dict]:
    """테이블의 외래 키 관계"""
    query = """
        SELECT
            kcu.column_name,
            ccu.table_name AS foreign_table,
            ccu.column_name AS foreign_column
        FROM information_schema.table_constraints tc
        JOIN information_schema.key_column_usage kcu
            ON tc.constraint_name = kcu.constraint_name
        JOIN information_schema.constraint_column_usage ccu
            ON tc.constraint_name = ccu.constraint_name
        WHERE tc.constraint_type = 'FOREIGN KEY'
          AND tc.table_name = $1
    """
    return [dict(r) for r in await pool.fetch(query, table_name)]

async def get_indexes(pool: asyncpg.Pool, table_name: str) -> list[dict]:
    """테이블의 인덱스 정보"""
    query = """
        SELECT indexname, indexdef
        FROM pg_indexes
        WHERE tablename = $1 AND schemaname = 'public'
    """
    return [dict(r) for r in await pool.fetch(query, table_name)]

def format_schema_markdown(
    table_name: str, 
    columns: list[dict],
    foreign_keys: list[dict],
    indexes: list[dict]
) -> str:
    """스키마 정보를 LLM 친화적 마크다운으로 포맷"""
    lines = [f"## {table_name}\n"]
    
    # 컬럼 테이블
    lines.append("### Columns")
    lines.append("| Column | Type | Nullable | Default |")
    lines.append("|--------|------|----------|---------|")
    for c in columns:
        nullable = "YES" if c["is_nullable"] == "YES" else "NO"
        default = str(c["column_default"] or "-")
        if len(default) > 30:
            default = default[:27] + "..."
        lines.append(f"| {c['column_name']} | {c['data_type']} | {nullable} | {default} |")
    
    # 외래 키
    if foreign_keys:
        lines.append("\n### Foreign Keys")
        for fk in foreign_keys:
            lines.append(f"- `{fk['column_name']}` → `{fk['foreign_table']}.{fk['foreign_column']}`")
    
    # 인덱스
    if indexes:
        lines.append("\n### Indexes")
        for idx in indexes:
            lines.append(f"- `{idx['indexname']}`")
    
    return "\n".join(lines)
```

### Step 4: 서버 본체 (server.py)

```python
# server.py — PostgreSQL MCP 서버 진입점
import os
import json
from contextlib import asynccontextmanager
from typing import AsyncIterator

import asyncpg
from mcp.server.fastmcp import FastMCP

from db import get_pool, close_pool
from schema_tools import (
    list_tables, get_columns, get_foreign_keys,
    get_indexes, format_schema_markdown
)

MAX_ROWS = int(os.getenv("MAX_ROWS", "500"))
READ_ONLY = os.getenv("READ_ONLY", "true").lower() == "true"


# ── Lifespan: 커넥션 풀 초기화/정리 ──
@asynccontextmanager
async def app_lifespan(server: FastMCP) -> AsyncIterator[dict]:
    """서버 시작 시 풀 생성, 종료 시 풀 해제"""
    pool = await get_pool()
    
    # 연결 테스트
    async with pool.acquire() as conn:
        version = await conn.fetchval("SELECT version()")
    print(f"Connected to PostgreSQL: {version[:60]}...")
    
    yield {"pool": pool}
    
    await close_pool()
    print("Connection pool closed.")


mcp = FastMCP(
    "postgresql-explorer",
    description="PostgreSQL database explorer with schema introspection",
    lifespan=app_lifespan,
)


# ── Resources: 스키마 정보 ──
@mcp.resource("schema://tables")
async def resource_tables() -> str:
    """List all tables in the public schema with estimated row counts."""
    pool = await get_pool()
    tables = await list_tables(pool)
    
    lines = ["# Database Tables\n"]
    lines.append("| Table | Type | Est. Rows |")
    lines.append("|-------|------|-----------|")
    for t in tables:
        rows = t.get("estimated_rows") or "?"
        lines.append(f"| {t['table_name']} | {t['table_type']} | {rows} |")
    return "\n".join(lines)


@mcp.resource("schema://tables/{table_name}")
async def resource_table_schema(table_name: str) -> str:
    """Get detailed schema for a specific table including columns, 
    foreign keys, and indexes."""
    pool = await get_pool()
    
    # 테이블 존재 확인
    exists = await pool.fetchval(
        "SELECT EXISTS(SELECT 1 FROM information_schema.tables "
        "WHERE table_schema='public' AND table_name=$1)", 
        table_name
    )
    if not exists:
        return f"Table '{table_name}' not found in public schema."
    
    columns = await get_columns(pool, table_name)
    fks = await get_foreign_keys(pool, table_name)
    indexes = await get_indexes(pool, table_name)
    
    return format_schema_markdown(table_name, columns, fks, indexes)


# ── Tools: 쿼리 실행 ──
@mcp.tool()
async def read_query(sql: str) -> str:
    """Execute a read-only SQL query against the PostgreSQL database.
    
    The query runs inside a READ ONLY transaction with REPEATABLE READ 
    isolation. Results are limited to MAX_ROWS rows. Only SELECT 
    statements are allowed.
    
    Args:
        sql: A valid SQL SELECT statement. Use $1, $2 etc. for 
             parameterized queries (parameters not yet supported 
             in this version).
    """
    pool = await get_pool()
    
    # 기본 검증: SELECT만 허용
    stripped = sql.strip().upper()
    if not stripped.startswith("SELECT") and not stripped.startswith("WITH"):
        return "Error: Only SELECT and WITH (CTE) queries are allowed."
    
    # LIMIT 강제 주입
    if "LIMIT" not in stripped:
        sql = f"SELECT * FROM ({sql}) _sub LIMIT {MAX_ROWS}"
    
    try:
        async with pool.acquire() as conn:
            async with conn.transaction(
                readonly=True, 
                isolation="repeatable_read"
            ):
                rows = await conn.fetch(sql)
                
                if not rows:
                    return "Query returned no results."
                
                # 마크다운 테이블로 포맷팅
                columns = list(rows[0].keys())
                lines = ["| " + " | ".join(columns) + " |"]
                lines.append("| " + " | ".join(["---"] * len(columns)) + " |")
                for row in rows[:MAX_ROWS]:
                    values = [str(row[c]) if row[c] is not None else "NULL" 
                             for c in columns]
                    lines.append("| " + " | ".join(values) + " |")
                
                result = "\n".join(lines)
                if len(rows) >= MAX_ROWS:
                    result += f"\n\n*Results truncated to {MAX_ROWS} rows.*"
                return result
                
    except asyncpg.PostgresError as e:
        return f"Query error: {e}"


@mcp.tool()
async def write_query(sql: str) -> str:
    """Execute a write SQL statement (INSERT, UPDATE, DELETE).
    
    Only available when READ_ONLY mode is disabled. Each statement
    runs in its own transaction with automatic commit on success
    or rollback on failure.
    
    Args:
        sql: A valid INSERT, UPDATE, or DELETE statement.
    """
    if READ_ONLY:
        return "Error: Server is in read-only mode. Write operations are disabled."
    
    pool = await get_pool()
    stripped = sql.strip().upper()
    
    # DDL 차단
    for keyword in ("DROP", "ALTER", "CREATE", "TRUNCATE", "GRANT", "REVOKE"):
        if stripped.startswith(keyword):
            return f"Error: {keyword} statements are not allowed."
    
    try:
        async with pool.acquire() as conn:
            async with conn.transaction():
                result = await conn.execute(sql)
                return f"Success: {result}"
    except asyncpg.PostgresError as e:
        return f"Write error: {e}"


@mcp.tool()
async def explain_query(sql: str) -> str:
    """Show the query execution plan using EXPLAIN ANALYZE.
    
    Useful for understanding query performance. The query runs
    inside a read-only transaction and is rolled back after analysis.
    
    Args:
        sql: The SQL query to analyze.
    """
    pool = await get_pool()
    
    try:
        async with pool.acquire() as conn:
            async with conn.transaction(readonly=True):
                rows = await conn.fetch(f"EXPLAIN ANALYZE {sql}")
                plan = "\n".join(row["QUERY PLAN"] for row in rows)
                return f"```\n{plan}\n```"
    except asyncpg.PostgresError as e:
        return f"Explain error: {e}"


@mcp.tool()
async def list_databases() -> str:
    """List all accessible databases on this PostgreSQL server."""
    pool = await get_pool()
    
    rows = await pool.fetch(
        "SELECT datname, pg_database_size(datname) AS size_bytes "
        "FROM pg_database WHERE datistemplate = false ORDER BY datname"
    )
    
    lines = ["| Database | Size |"]
    lines.append("|----------|------|")
    for r in rows:
        size_mb = r["size_bytes"] / (1024 * 1024)
        lines.append(f"| {r['datname']} | {size_mb:.1f} MB |")
    return "\n".join(lines)


# ── 진입점 ──
if __name__ == "__main__":
    mcp.run(transport="stdio")
```

### Step 5: 실행 및 테스트

```run:python
# 실행 명령어와 Claude Desktop 설정 출력
import json

# 서버 실행
print("# 직접 실행:")
print("  uv run server.py\n")

# Claude Desktop 설정
config = {
    "mcpServers": {
        "postgresql-explorer": {
            "command": "uv",
            "args": ["run", "server.py"],
            "env": {
                "DATABASE_URL": "postgresql://user:pass@localhost:5432/mydb",
                "READ_ONLY": "true",
                "MAX_ROWS": "500"
            }
        }
    }
}

print("# Claude Desktop 설정 (claude_desktop_config.json):")
print(json.dumps(config, indent=2, ensure_ascii=False))
```

```output
# 직접 실행:
  uv run server.py

# Claude Desktop 설정 (claude_desktop_config.json):
{
  "mcpServers": {
    "postgresql-explorer": {
      "command": "uv",
      "args": ["run", "server.py"],
      "env": {
        "DATABASE_URL": "postgresql://user:pass@localhost:5432/mydb",
        "READ_ONLY": "true",
        "MAX_ROWS": "500"
      }
    }
  }
}
```

MCP Inspector로 테스트하는 것도 잊지 마세요. [Inspector 사용법](03-ch3-개발-환경-설정과-첫-mcp-서버/04-04-mcp-inspector와-디버깅.md)에서 배운 것처럼:

```bash
# MCP Inspector로 대화형 테스트
npx @modelcontextprotocol/inspector uv run server.py
```

## 더 깊이 알아보기

### asyncpg의 탄생 — "psycopg가 있는데 왜?"

asyncpg는 MagicStack의 Yury Selivanov가 2016년에 만들었습니다. Yury는 Python의 `async/await` 문법을 설계한 핵심 인물(PEP 492 저자)이기도 하죠. 그는 기존 PostgreSQL 드라이버(psycopg2)가 libpq C 라이브러리 위에 구축되어 있어 asyncio와 자연스럽게 통합되지 못하는 문제를 해결하고 싶었습니다.

asyncpg의 혁신적 접근은 **libpq를 완전히 우회**한 것입니다. PostgreSQL의 바이너리 프로토콜을 Cython으로 직접 구현하여 중간 계층 없이 데이터를 주고받습니다. 이 결정 덕분에 psycopg3 대비 약 5배, 텍스트 프로토콜 기반 드라이버 대비 10배 이상의 성능을 달성했습니다.

재미있는 점은, asyncpg의 파라미터 바인딩이 `$1, $2` 형식을 사용하는 이유도 여기에 있습니다. 이것은 PostgreSQL 서버가 내부적으로 사용하는 **네이티브 파라미터 형식** 그대로입니다. psycopg의 `%s`나 SQLite의 `?`는 드라이버가 중간에서 변환하는 것이지만, asyncpg는 직접 통신하므로 변환 단계가 필요 없는 거죠.

### 레퍼런스 서버의 이주 — servers에서 servers-archived로

2025년 말, Anthropic은 공식 레퍼런스 서버 저장소를 `modelcontextprotocol/servers`에서 `modelcontextprotocol/servers-archived`로 이전했습니다. 이유가 흥미롭습니다. MCP 생태계가 성장하면서, 커뮤니티 서버의 품질이 레퍼런스 서버를 넘어서는 경우가 늘어난 거죠. Anthropic은 "우리가 모든 DB 서버를 유지하는 것보다, 커뮤니티가 더 나은 구현을 만들도록 돕는 게 낫다"고 판단했습니다.

그래서 레퍼런스 서버는 이제 **교육용 참고 자료**로 남아 있고, 프로덕션 용도로는 Postgres MCP Pro, pg-mcp-server 같은 커뮤니티 프로젝트를 권장합니다. 이것은 오픈소스 생태계의 건강한 성숙을 보여주는 사례입니다.

## 흔한 오해와 팁

> ⚠️ **흔한 오해**: "asyncpg가 빠르니까 무조건 asyncpg를 써야 한다"고 생각하기 쉽습니다. 하지만 asyncpg는 `$1` 파라미터 형식만 지원하고, Django/SQLAlchemy ORM과의 통합이 제한적입니다. 기존 프로젝트에 MCP 서버를 추가할 때는 psycopg3(`psycopg[binary]`)이 더 실용적일 수 있습니다. psycopg3도 asyncio를 지원하며 `%s` 형식과 `sql.Identifier()`로 안전한 동적 쿼리를 지원합니다.

> 💡 **알고 계셨나요?**: PostgreSQL의 `information_schema`는 SQL:2003 표준에 정의되어 있어서 MySQL, SQL Server 등 다른 RDBMS에서도 비슷하게 동작합니다. 하지만 PostgreSQL 고유의 `pg_catalog` 시스템 카탈로그는 더 풍부한 정보(인덱스 상세, 테이블 크기, 통계 등)를 제공합니다. 프로덕션 MCP 서버라면 `information_schema`와 `pg_catalog`를 함께 활용하세요.

> 🔥 **실무 팁**: asyncpg 커넥션 풀의 `max_queries=50000` 설정은 커넥션 "회전"을 의미합니다. 5만 번 쿼리한 커넥션은 자동으로 폐기하고 새 커넥션으로 교체합니다. 이 메커니즘은 PostgreSQL의 메모리 누수(특히 prepared statement 캐시 증가)를 방지합니다. 프로덕션에서 `max_queries`를 너무 낮게 잡으면 불필요한 재연결이 발생하고, 너무 높게 잡으면 메모리 문제가 생길 수 있습니다. 5만~10만이 적정 범위입니다.

## 핵심 정리

| 개념 | 설명 |
|------|------|
| 공식 레퍼런스 서버 | TypeScript 초기 버전 기준 약 130줄. query 도구 1개 + 스키마 Resource. READ ONLY + ROLLBACK으로 안전성 보장 |
| asyncpg 커넥션 풀 | `create_pool(min_size, max_size)`. TCP 핸드셰이크 비용 제거. `pool.acquire()` 컨텍스트 매니저 |
| 파라미터 바인딩 | asyncpg는 `$1, $2` (PostgreSQL 네이티브). SQLite의 `?`와 다름 |
| 트랜잭션 격리 | `readonly=True` + `isolation="repeatable_read"`로 일관된 읽기 보장 |
| 스키마 인트로스펙션 | `information_schema.columns/tables/constraints`를 Resource로 노출 |
| 다중 DB 지원 | PoolManager 패턴(asyncpg.Pool 래퍼) — db_id로 풀을 관리, LLM이 대상 DB 선택 |
| 토큰 효율 | 마크다운 테이블 > JSON (30~40% 절약). TSV가 최적이나 가독성 트레이드오프 |
| 레퍼런스 서버 한계 | SQL 인젝션 방어/행 수 제한/감사 로깅 부재 → 프로덕션은 보안 계층 추가 필수 |

## 다음 섹션 미리보기

지금까지 설계 전략(7.1), SQLite 구현(7.2), 보안 계층(7.3), PostgreSQL과 레퍼런스 분석(7.4)을 다뤘습니다. 마지막 세션 [E-Commerce 데이터 서버 실습](07-ch7-실전-서버-데이터베이스-연동/05-05-e-commerce-데이터-서버-실습.md)에서는 이 모든 것을 통합합니다. 상품, 주문, 고객 데이터가 있는 실제 e-commerce 데이터베이스를 대상으로, 스키마 Resource + 안전한 쿼리 Tool + 인사이트 Prompt를 갖춘 **프로덕션급 MCP 서버**를 처음부터 끝까지 구축합니다.

## 참고 자료

- [MCP Official Reference Servers (Archived)](https://github.com/modelcontextprotocol/servers-archived/tree/main/src/postgres) - Anthropic 공식 PostgreSQL 레퍼런스 서버 소스 코드. 초기 버전 약 130줄로 MCP DB 서버의 정수를 보여줌
- [asyncpg Documentation](https://magicstack.github.io/asyncpg/current/) - asyncpg 공식 문서. 커넥션 풀 API, 타입 코덱, 트랜잭션 관리 상세 설명
- [Build an MCP Server with Python — Tech Insider](https://tech-insider.org/how-to-build-mcp-server-python-fastmcp-tutorial/) - FastMCP + asyncpg 12단계 튜토리얼. 커넥션 풀링부터 Docker 배포까지 프로덕션급 코드
- [Lessons Learned Writing an MCP Server for PostgreSQL — pgEdge](https://www.pgedge.com/blog/lessons-learned-writing-an-mcp-server-for-postgresql) - TSV 토큰 효율, progressive disclosure, LIMIT 전략 등 실전 운영 인사이트
- [MCP Python SDK (GitHub)](https://github.com/modelcontextprotocol/python-sdk) - FastMCP가 통합된 공식 Python SDK. lifespan 패턴, Resource/Tool 데코레이터 레퍼런스
- [Postgres MCP Pro — Crystal DBA](https://github.com/crystaldba/postgres-mcp) - explain_query, analyze_db_health 등 9개 도구를 갖춘 프로덕션급 커뮤니티 서버

---
### 🔗 Related Sessions
- [appcontext](07-ch7-실전-서버-데이터베이스-연동/02-02-sqlite-mcp-서버-구축.md) (prerequisite)
- [securequeryexecutor](07-ch7-실전-서버-데이터베이스-연동/03-03-쿼리-안전성과-권한-제어.md) (prerequisite)
