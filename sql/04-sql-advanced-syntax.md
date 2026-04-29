# SQL Advanced Syntax

기본 문법 (`SELECT`, `WHERE`, `JOIN`, `GROUP BY`) 만으로도 많은 일을 할 수 있지만, 데이터가 복잡해지면 **서브쿼리 (Subquery)**, **CTE (`WITH`)**, **윈도우 함수 (Window Function)** 가 진가를 발휘한다. 이 세 가지는 모두 "쿼리 안에서 쿼리를 쓰거나, 행을 그룹과 행 둘 다로 다루는" 것을 가능하게 한다.

---

## 서브쿼리 (Subquery)

쿼리 안에 들어있는 또 다른 쿼리. **사용 위치**에 따라 종류가 나뉜다.

### 1. Scalar Subquery — 단일 값을 반환

`SELECT` 절이나 `WHERE` 절에서 하나의 값처럼 사용한다.

```sql
-- Each user with the global average age
SELECT
  name,
  age,
  (SELECT AVG(age) FROM users) AS avg_age
FROM users;
```

```sql
-- Users older than the average
SELECT name, age
FROM users
WHERE age > (SELECT AVG(age) FROM users);
```

> 스칼라 서브쿼리는 정확히 1개의 값을 반환해야 한다. 여러 행이 나오면 에러.

### 2. Inline View (FROM 절 서브쿼리) — 임시 테이블처럼

`FROM` 절에 서브쿼리를 두면 그 결과가 **임시 테이블** 처럼 동작한다.

```sql
SELECT country, avg_age
FROM (
  SELECT country, AVG(age) AS avg_age
  FROM users
  GROUP BY country
) AS country_stats
WHERE avg_age >= 25;
```

별칭 (위 예제의 `country_stats`) 은 **필수**다.

### 3. IN / NOT IN — 여러 값과 비교

```sql
-- Users who have at least one order
SELECT name
FROM users
WHERE id IN (SELECT user_id FROM orders);

-- Users who have NO order
SELECT name
FROM users
WHERE id NOT IN (SELECT user_id FROM orders);
```

> ⚠️ `NOT IN`의 함정: 서브쿼리 결과에 **`NULL`이 하나라도 있으면** 전체 결과가 비게 된다 (3값 논리 때문). 안전하게 쓰려면 `NOT EXISTS`를 권장.

### 4. EXISTS / NOT EXISTS — 존재 여부

```sql
-- Users who have at least one order (safer than IN)
SELECT name
FROM users u
WHERE EXISTS (
  SELECT 1 FROM orders o WHERE o.user_id = u.id
);

-- Users with no order (NULL-safe alternative to NOT IN)
SELECT name
FROM users u
WHERE NOT EXISTS (
  SELECT 1 FROM orders o WHERE o.user_id = u.id
);
```

> `EXISTS`는 첫 매치를 찾으면 즉시 종료하므로 `IN` 보다 효율적인 경우가 많다 (옵티마이저가 보통 동등하게 변환하긴 한다).

### 5. Correlated Subquery — 상관 서브쿼리

서브쿼리가 **외부 쿼리의 행을 참조**하는 경우. 외부 행마다 서브쿼리가 (논리적으로) 한 번씩 실행된다.

```sql
-- Users whose age is above their country's average
SELECT u.name, u.age, u.country
FROM users u
WHERE u.age > (
  SELECT AVG(age)
  FROM users
  WHERE country = u.country   -- references outer u
);
```

상관 서브쿼리는 강력하지만 비싸다. 다음에 나올 **윈도우 함수**로 대체할 수 있는 경우가 많다.

---

## CTE — Common Table Expression (`WITH`)

### CTE란?

**CTE**는 쿼리 앞에 임시로 이름 붙인 결과 집합을 만들어두고, 메인 쿼리에서 **테이블처럼 참조**하는 문법이다. SQL:1999 표준에서 도입됐다.

```sql
WITH country_stats AS (
  SELECT country, AVG(age) AS avg_age, COUNT(*) AS user_count
  FROM users
  GROUP BY country
)
SELECT *
FROM country_stats
WHERE user_count >= 3
ORDER BY avg_age DESC;
```

### 왜 CTE를 쓰는가?

#### (1) 가독성

서브쿼리를 인라인으로 깊게 중첩하면 읽기 힘들다.

```sql
-- ❌ Nested subqueries — hard to read
SELECT *
FROM (
  SELECT country, avg_age
  FROM (
    SELECT country, AVG(age) AS avg_age, COUNT(*) AS user_count
    FROM users
    GROUP BY country
  ) AS s
  WHERE user_count >= 3
) AS final
ORDER BY avg_age DESC;

-- ✅ With CTE — read top to bottom
WITH country_stats AS (
  SELECT country, AVG(age) AS avg_age, COUNT(*) AS user_count
  FROM users
  GROUP BY country
),
big_countries AS (
  SELECT * FROM country_stats WHERE user_count >= 3
)
SELECT *
FROM big_countries
ORDER BY avg_age DESC;
```

#### (2) 재사용

같은 서브쿼리를 여러 곳에서 참조할 때 한 번만 정의.

```sql
WITH active_users AS (
  SELECT id FROM users WHERE last_login > NOW() - INTERVAL '30 days'
)
SELECT
  (SELECT COUNT(*) FROM active_users)                     AS active_count,
  (SELECT COUNT(*) FROM orders WHERE user_id IN
       (SELECT id FROM active_users))                     AS active_orders;
```

#### (3) 재귀 — 계층 데이터

`WITH RECURSIVE`로 트리/그래프 같은 계층 구조를 풀 수 있다.

```sql
-- Categories table: id, name, parent_id
-- Find all descendants of category 1
WITH RECURSIVE descendants AS (
  -- Anchor: starting row(s)
  SELECT id, name, parent_id, 0 AS depth
  FROM categories
  WHERE id = 1

  UNION ALL

  -- Recursive: rows that reference the previous level
  SELECT c.id, c.name, c.parent_id, d.depth + 1
  FROM categories c
  JOIN descendants d ON c.parent_id = d.id
)
SELECT * FROM descendants ORDER BY depth, id;
```

활용 사례:
- 조직도 / 카테고리 트리
- 댓글 스레드 (parent-child)
- 그래프 탐색 (예: "친구의 친구")
- 숫자/날짜 시퀀스 생성

> ⚠️ 종료 조건이 없으면 무한 루프. 일부 DBMS는 `MAXRECURSION` 같은 안전장치를 제공한다.

### CTE vs Subquery — 어느 것을 쓸까?

| 항목 | Subquery | CTE |
| --- | --- | --- |
| 가독성 | 짧으면 OK, 깊으면 나쁨 | 항상 좋음 |
| 재사용 | 매번 다시 씀 | 한 번 정의, 여러 번 참조 |
| 재귀 | 불가 | `WITH RECURSIVE` 가능 |
| 성능 | 옵티마이저가 잘 합쳐줌 | DBMS에 따라 다름 (PostgreSQL 12+ 부터는 inline 됨) |

> 옛 PostgreSQL (12 이전) 에서는 CTE가 **optimization fence** 역할을 해서 옵티마이저가 안쪽으로 들어가지 못했다. 12 이후로는 자동으로 inline 된다.

---

## Window Function — 윈도우 함수

### 윈도우 함수란?

> "Compute a value across a *window* of rows related to the current row, **without collapsing the rows**."

집계 함수 (`SUM`, `AVG`, `COUNT` ...) 는 그룹을 하나의 행으로 압축한다. **윈도우 함수**는 같은 계산을 하되 **각 행을 그대로 유지**하면서 그 행 옆에 결과를 덧붙인다.

```sql
-- Aggregate (collapses rows)
SELECT country, AVG(age) FROM users GROUP BY country;
-- → one row per country

-- Window function (keeps every row)
SELECT name, country, age,
       AVG(age) OVER (PARTITION BY country) AS country_avg_age
FROM users;
-- → still one row per user, with extra column
```

### OVER 절 — 윈도우 정의

윈도우 함수의 핵심은 **`OVER`** 절. 이 안에서 "어떤 범위의 행을 보고 계산할지"를 정의한다.

```sql
function_name(args) OVER (
  PARTITION BY  ...   -- 그룹 (없으면 전체 한 그룹)
  ORDER BY      ...   -- 정렬 (랭킹/누적 함수에 필요)
  ROWS / RANGE  ...   -- 윈도우 프레임 (이동 평균 등)
)
```

### 주요 윈도우 함수

#### 집계 윈도우 함수

```sql
SELECT
  name, country, age,
  COUNT(*)  OVER (PARTITION BY country)            AS users_in_country,
  AVG(age)  OVER (PARTITION BY country)            AS avg_age_in_country,
  SUM(age)  OVER (PARTITION BY country)            AS total_age_in_country,
  age - AVG(age) OVER (PARTITION BY country)       AS diff_from_avg
FROM users;
```

#### 랭킹 함수

```sql
SELECT
  name, country, age,
  ROW_NUMBER() OVER (PARTITION BY country ORDER BY age DESC) AS rn,
  RANK()       OVER (PARTITION BY country ORDER BY age DESC) AS rnk,
  DENSE_RANK() OVER (PARTITION BY country ORDER BY age DESC) AS drnk,
  NTILE(4)     OVER (ORDER BY age)                           AS quartile
FROM users;
```

| 함수 | 동점 처리 |
| --- | --- |
| `ROW_NUMBER()` | 1, 2, 3, 4 — 항상 유니크 |
| `RANK()` | 1, 2, **2, 4** — 동점 후 점프 |
| `DENSE_RANK()` | 1, 2, **2, 3** — 동점이 있어도 다음은 +1 |
| `NTILE(n)` | n개 버킷으로 균등 분할 (분위수) |

#### 오프셋 함수 — 다른 행 참조

```sql
SELECT
  user_id, created_at, amount,
  LAG(amount)  OVER (PARTITION BY user_id ORDER BY created_at) AS prev_amount,
  LEAD(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS next_amount,
  amount - LAG(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS diff_from_prev
FROM orders;
```

`LAG(col, n)` — n 행 앞의 값, `LEAD(col, n)` — n 행 뒤의 값. 시계열 분석에 필수.

#### 첫/마지막 값

```sql
SELECT
  user_id, created_at, amount,
  FIRST_VALUE(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS first_order_amount,
  LAST_VALUE(amount)  OVER (
    PARTITION BY user_id ORDER BY created_at
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  )                                                                   AS last_order_amount
FROM orders;
```

> `LAST_VALUE`는 기본 프레임이 "현재 행까지" 라서 의도와 다르게 동작한다. 위처럼 명시적 프레임을 줘야 한다.

### Window Frame — 프레임 정의

`ORDER BY`만 있는 윈도우는 기본 프레임이 `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — 즉 **누적 (running total)** 의 범위.

```sql
-- Running total of orders per user
SELECT
  user_id, created_at, amount,
  SUM(amount) OVER (
    PARTITION BY user_id
    ORDER BY created_at
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS running_total,

  -- 7-row moving average (current + 6 previous)
  AVG(amount) OVER (
    PARTITION BY user_id
    ORDER BY created_at
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) AS moving_avg_7
FROM orders;
```

프레임 옵션:
- `ROWS` — 행 개수 기준 (정확)
- `RANGE` — 값 범위 기준 (`ORDER BY` 컬럼 값으로)
- `GROUPS` — 동일 정렬 키를 가진 그룹 단위 (덜 흔함)

경계:
- `UNBOUNDED PRECEDING` — 파티션 시작
- `n PRECEDING` — n행 앞
- `CURRENT ROW` — 현재 행
- `n FOLLOWING` — n행 뒤
- `UNBOUNDED FOLLOWING` — 파티션 끝

### 윈도우 함수 vs GROUP BY

| 항목 | GROUP BY | Window Function |
| --- | --- | --- |
| 결과 행 수 | 그룹 수만큼 축소 | 입력과 동일 |
| 원본 컬럼 노출 | 그룹 컬럼 + 집계만 | 모든 컬럼 OK |
| 동시 사용 | "각 그룹 1행" | "각 행 옆에 그룹 통계" |

```sql
-- "각 사용자별 주문 합계만 필요" → GROUP BY
SELECT user_id, SUM(amount) FROM orders GROUP BY user_id;

-- "각 주문 행 옆에 그 사용자의 누적/총합도 같이 보고 싶다" → Window
SELECT user_id, created_at, amount,
       SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS running_total,
       SUM(amount) OVER (PARTITION BY user_id) AS user_total
FROM orders;
```

---

## 자주 나오는 패턴 모음

### Top-N per Group — 그룹별 상위 N개

> "각 country에서 가장 나이 많은 사용자 1명씩"

```sql
WITH ranked AS (
  SELECT name, country, age,
         ROW_NUMBER() OVER (PARTITION BY country ORDER BY age DESC) AS rn
  FROM users
)
SELECT name, country, age
FROM ranked
WHERE rn = 1;
```

### Running Total — 누적 합계

```sql
SELECT user_id, created_at, amount,
       SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS running_total
FROM orders;
```

### Period-over-Period — 이전 기간 비교

```sql
SELECT
  month,
  revenue,
  LAG(revenue) OVER (ORDER BY month) AS prev_revenue,
  revenue - LAG(revenue) OVER (ORDER BY month) AS diff,
  (revenue - LAG(revenue) OVER (ORDER BY month))
    / LAG(revenue) OVER (ORDER BY month) * 100 AS pct_change
FROM monthly_revenue;
```

### Median (중앙값) — `PERCENTILE_CONT`

```sql
SELECT country,
       PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY age) AS median_age
FROM users
GROUP BY country;
```

### 누적 분포 — `CUME_DIST`, `PERCENT_RANK`

```sql
SELECT name, age,
       CUME_DIST()    OVER (ORDER BY age) AS cumulative_distribution,
       PERCENT_RANK() OVER (ORDER BY age) AS percent_rank
FROM users;
```

---

## 정리

- **Subquery**: 위치별 5가지 — Scalar / Inline / IN / EXISTS / Correlated
- `NOT IN`은 `NULL`에 취약하므로 `NOT EXISTS` 권장
- **CTE (`WITH`)**: 가독성, 재사용, 재귀 (`WITH RECURSIVE`) 의 세 가지 무기
- **Window Function**: "행을 줄이지 않고" 그룹 통계를 붙이는 도구 — `OVER (PARTITION BY ... ORDER BY ...)`
- 랭킹 (`ROW_NUMBER`/`RANK`/`DENSE_RANK`), 오프셋 (`LAG`/`LEAD`), 누적/이동 (`SUM` + frame) 이 핵심
- 상관 서브쿼리로 짜던 패턴들이 윈도우 함수로 더 깔끔/효율적으로 표현되는 경우가 많다

이제 SQL 문법은 충분히 정리되었다. 다음 문서 (`05-query-engines.md`) 에서는 우리가 작성한 SQL을 **실제로 실행하는 엔진들** — 특히 **Trino** 같은 분산 쿼리 엔진의 세계를 살펴본다.
