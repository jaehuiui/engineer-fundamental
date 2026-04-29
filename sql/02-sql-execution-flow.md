# SQL Execution Flow

## 핵심 질문: SQL은 어떻게 "해석"되고 "실행"되는가?

우리가 `SELECT name FROM users WHERE age >= 20;` 한 줄을 보내면, 데이터베이스는 이걸 어떻게 처리할까? SQL이 **선언형 언어**라는 점 (참고: `01-sql-fundamentals.md`) 을 떠올리면, "무엇을" 만 적혀있는 텍스트를 "어떻게 실행할지" 로 변환하는 과정이 반드시 필요하다.

이 변환 과정 전체를 **Query Processing Pipeline** 이라고 부른다.

---

## Query Processing Pipeline

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   SQL Text   │───▶│    Parser    │───▶│   Binder /   │───▶│   Optimizer  │───▶│   Executor   │
│ "SELECT ..." │    │ (Lex/Parse)  │    │   Analyzer   │    │ (Plan Maker) │    │ (Run Plan)   │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
                          │                    │                    │                    │
                          ▼                    ▼                    ▼                    ▼
                    Parse Tree (AST)    Validated AST       Execution Plan         Result Set
```

각 단계를 하나씩 보자.

### 1. Parser (구문 분석)

입력된 SQL **문자열**을 **트리 구조 (Abstract Syntax Tree, AST)** 로 변환한다.

- **Lexer (Tokenizer)**: 문자열을 의미 있는 단위 (토큰) 로 쪼갬
  - `SELECT name FROM users` → `[SELECT, name, FROM, users]`
- **Parser**: 토큰들을 SQL 문법 규칙에 따라 트리로 조립
  - 문법이 틀리면 여기서 **Syntax Error** 발생

```
SelectStatement
├── SelectList: [name]
├── FromClause: users
└── WhereClause:
    └── Comparison: age >= 20
```

> 이 단계까지는 "테이블이 진짜 있는지", "컬럼 이름이 맞는지"는 모른다. 순수하게 문법만 본다.

### 2. Binder / Analyzer (의미 분석)

AST에 등장하는 **이름 (테이블, 컬럼, 함수)** 이 실제로 존재하는지를 데이터베이스의 **Catalog (메타데이터 저장소)** 와 대조하여 검증한다.

확인하는 것들:
- 테이블 `users`가 존재하는가?
- 컬럼 `name`, `age`가 그 테이블에 있는가?
- 사용된 함수가 존재하는가? (`SUM`, `COUNT` 등)
- 데이터 타입이 호환되는가? (`age >= 'twenty'` 같은 케이스)
- 사용자에게 이 테이블에 대한 **권한 (Privilege)** 이 있는가?

문제가 있으면 **Semantic Error** (예: `column "namee" does not exist`).

### 3. Optimizer (최적화) — 가장 중요한 단계

이게 **선언형 언어의 핵심**이다. 동일한 결과를 내는 실행 방법은 여러 가지가 있는데, 그 중 가장 빠른 (또는 비용이 낮은) 방법을 골라내는 단계.

#### 옵티마이저의 두 가지 접근

**Rule-Based Optimizer (RBO, 규칙 기반)**
- 미리 정해진 규칙으로 실행 계획을 결정
- 예: "WHERE 절은 가능한 한 일찍 적용", "인덱스가 있으면 인덱스 사용"
- 단순하지만 통계 정보를 활용하지 못함
- Oracle 9i 이후로는 거의 사용되지 않음

**Cost-Based Optimizer (CBO, 비용 기반)**
- **테이블 통계 (Table Statistics)** 를 기반으로 여러 후보 계획의 **예상 비용 (Estimated Cost)** 을 계산하여 가장 저렴한 것을 선택
- 통계 정보 예시:
  - 테이블의 행 수 (Row count)
  - 컬럼의 카디널리티 (Cardinality, 고유값 개수)
  - 데이터 분포 (Histogram)
  - 인덱스 정보
- **현대 RDBMS의 기본 옵티마이저**는 모두 CBO

#### 옵티마이저가 고려하는 결정들

```sql
SELECT u.name, COUNT(o.id)
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.country = 'KR'
GROUP BY u.name;
```

같은 결과를 내는 여러 실행 계획:

1. **JOIN 순서**: `users → orders` vs `orders → users`
2. **JOIN 알고리즘**: Nested Loop Join, Hash Join, Sort-Merge Join 중 어떤 것?
3. **인덱스 사용**: `users.country` 인덱스를 쓸까, Full Scan?
4. **필터 위치**: `WHERE country = 'KR'` 을 JOIN 전에 적용 (Predicate Pushdown)
5. **집계 방식**: Hash Aggregate vs Sort Aggregate

옵티마이저는 이 모든 조합 중에서 비용이 최소인 것을 선택한다.

### 4. Executor (실행)

Optimizer가 만든 **Execution Plan (실행 계획)** 을 받아 실제로 실행한다.

- 디스크에서 데이터를 읽고 (혹은 캐시 활용)
- JOIN, 필터, 집계, 정렬 등을 수행
- 결과를 클라이언트에게 반환

---

## Logical Query Processing Order (논리적 실행 순서)

여기서 정말 중요한 포인트. **SQL을 작성하는 순서와 실제로 처리되는 순서가 다르다.**

### 작성 순서 (우리가 쓰는 순서)

```sql
SELECT     column_list
FROM       table
JOIN       another_table ON ...
WHERE      filter_condition
GROUP BY   group_columns
HAVING     group_filter
ORDER BY   sort_columns
LIMIT      n;
```

### 논리적 처리 순서 (DB가 평가하는 순서)

```
1. FROM       — 어떤 테이블에서 가져올지
2. JOIN       — 다른 테이블과 결합
3. WHERE      — 행 단위로 필터링
4. GROUP BY   — 그룹화
5. HAVING     — 그룹 단위로 필터링
6. SELECT     — 가져올 컬럼/표현식 계산
7. DISTINCT   — 중복 제거
8. ORDER BY   — 정렬
9. LIMIT      — 개수 제한
```

> **"논리적"** 이라고 부르는 이유: 옵티마이저는 결과만 같으면 실제 실행은 자유롭게 재배치할 수 있다. 하지만 **결과의 의미**를 이해할 때는 이 순서를 기준으로 사고해야 한다.

### 왜 이 순서가 중요한가?

대표적으로 자주 헷갈리는 케이스:

#### 케이스 1: SELECT의 별칭 (alias) 을 WHERE에서 못 쓴다

```sql
-- ❌ Error: alias `senior` is unknown to WHERE
SELECT age >= 60 AS senior
FROM users
WHERE senior = TRUE;

-- ✅ WHERE runs before SELECT — use the original expression
SELECT age >= 60 AS senior
FROM users
WHERE age >= 60;
```

이유: `WHERE`(3단계)가 `SELECT`(6단계)보다 먼저 실행되므로, `SELECT`에서 정의한 별칭은 아직 존재하지 않는다.

#### 케이스 2: HAVING과 WHERE의 차이

```sql
-- WHERE: filters individual rows BEFORE grouping
-- HAVING: filters groups AFTER aggregation
SELECT country, COUNT(*) AS user_count
FROM users
WHERE age >= 20            -- (3) filter rows
GROUP BY country           -- (4) group
HAVING COUNT(*) > 100;     -- (5) filter groups
```

집계 함수 (`COUNT`, `SUM`, ...) 는 그룹화된 결과에만 의미가 있으므로 `WHERE`에서는 사용할 수 없다.

#### 케이스 3: ORDER BY는 SELECT의 별칭을 쓸 수 있다

```sql
-- ✅ ORDER BY runs AFTER SELECT, so alias is visible
SELECT name, age * 12 AS age_in_months
FROM users
ORDER BY age_in_months DESC;
```

---

## EXPLAIN — 실행 계획 들여다보기

옵티마이저가 만든 실행 계획을 보고 싶다면 `EXPLAIN`을 사용한다.

```sql
EXPLAIN SELECT * FROM users WHERE email = 'a@b.com';
```

PostgreSQL 출력 예시:

```
Index Scan using users_email_idx on users  (cost=0.42..8.44 rows=1 width=64)
  Index Cond: (email = 'a@b.com'::text)
```

읽는 법:
- **Index Scan**: 인덱스를 사용해서 조회 (좋음)
- **cost=0.42..8.44**: 시작 비용..전체 비용 (단위: 임의의 옵티마이저 단위)
- **rows=1**: 예상 행 수
- **width=64**: 평균 행 크기 (bytes)

### EXPLAIN ANALYZE — 실제로 실행하고 측정

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'a@b.com';
```

`ANALYZE`를 붙이면 **실제로 쿼리를 실행**하고 **실측 시간**까지 보여준다.

```
Index Scan using users_email_idx on users
  (cost=0.42..8.44 rows=1 width=64)
  (actual time=0.025..0.027 rows=1 loops=1)
Planning Time: 0.123 ms
Execution Time: 0.045 ms
```

> 옵티마이저의 **예상 (estimated)** 과 **실제 (actual)** 가 크게 차이나면 → 통계가 오래되었거나 (`ANALYZE` 명령으로 갱신 필요) 데이터 분포가 옵티마이저의 가정과 다른 경우다.

### 자주 보는 Plan Node 종류

| Node | 의미 | 좋고 나쁨 |
| --- | --- | --- |
| **Seq Scan** | 테이블 전체 스캔 | 작은 테이블이거나 대부분 행을 읽을 때만 OK |
| **Index Scan** | 인덱스로 행 위치 찾고 테이블 접근 | 일반적으로 좋음 |
| **Index Only Scan** | 인덱스에서만 데이터 추출 (테이블 접근 X) | 매우 좋음 |
| **Bitmap Index Scan** | 인덱스로 비트맵 만들고 일괄 접근 | 중간 정도 결과 행 수에 유리 |
| **Nested Loop** | 작은 테이블 × 인덱스 있는 큰 테이블 | 작은 입력에 유리 |
| **Hash Join** | 한쪽으로 해시 테이블 만들고 조인 | 큰 데이터끼리 동등 조인 |
| **Merge Join** | 양쪽 정렬 후 머지 | 정렬 비용 vs 큰 데이터에 유리 |
| **Sort** | 정렬 | 비싸므로 인덱스로 대체 가능한지 검토 |

---

## 인덱스가 실행에 미치는 영향 (간단히)

`WHERE`, `JOIN`, `ORDER BY` 등에서 인덱스가 있는 컬럼을 사용하면 옵티마이저가 **Index Scan**을 선택할 수 있다.

```sql
-- Without index: Sequential Scan (read whole table)
SELECT * FROM users WHERE email = 'a@b.com';

-- After creating an index
CREATE INDEX idx_users_email ON users(email);

-- Now: Index Scan (jump straight to the row)
SELECT * FROM users WHERE email = 'a@b.com';
```

대부분의 RDBMS는 **B-Tree 인덱스**를 기본으로 사용한다 (다른 종류: Hash, GiST, GIN, BRIN 등).

> 인덱스는 만능이 아니다 — `INSERT`/`UPDATE`/`DELETE` 시 인덱스도 함께 갱신해야 하므로 쓰기 비용이 증가한다. 또한 인덱스도 디스크 공간을 차지한다.

---

## Prepared Statement — 한 번 만들고 여러 번 실행

같은 쿼리를 **값만 바꿔서** 반복 실행할 때, 매번 Parse → Bind → Optimize 를 다시 하는 건 낭비다.

```sql
-- Prepare once
PREPARE find_user (text) AS
  SELECT * FROM users WHERE email = $1;

-- Execute many times with different parameters
EXECUTE find_user('a@b.com');
EXECUTE find_user('c@d.com');
```

이게 **SQL Injection 방어**의 본질이기도 하다 — 파라미터는 값으로만 바인딩되어 SQL 구문으로 해석되지 않는다.

---

## 정리

- SQL 실행은 **Parser → Binder → Optimizer → Executor** 의 4단계 파이프라인
- 옵티마이저 (특히 **CBO**) 가 통계를 기반으로 최적 실행 계획을 선택
- 작성 순서와 다른 **논리적 실행 순서** 를 이해하는 것이 디버깅의 핵심
  - `FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT`
- **EXPLAIN / EXPLAIN ANALYZE** 로 실행 계획을 확인할 수 있다
- 인덱스, Prepared Statement는 실행 효율을 좌우하는 중요한 도구

다음 문서부터는 실제 SQL 문법 (`SELECT`, `WHERE`, `JOIN` 등) 을 정리한다.
