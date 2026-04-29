# SQL Syntax Basics

기본 데이터 조회 (`SELECT`, `WHERE`, `ORDER BY`, ...) 부터 집계 (`GROUP BY`, `HAVING`) 와 조인 (`JOIN`) 까지, 실무에서 가장 많이 쓰는 문법을 정리한다.

> 실행 순서가 헷갈리면 `02-sql-execution-flow.md`의 "Logical Query Processing Order" 섹션을 먼저 보자.

---

## 예제 데이터

문서 전반에서 사용할 예시 테이블.

```sql
-- users
+----+---------+---------+----------+
| id |  name   | country | age      |
+----+---------+---------+----------+
| 1  | Alice   | KR      | 25       |
| 2  | Bob     | US      | 30       |
| 3  | Carol   | KR      | 22       |
| 4  | Dave    | JP      | 40       |
| 5  | Eve     | KR      | 35       |
+----+---------+---------+----------+

-- orders
+----+---------+--------+------------+
| id | user_id | amount | created_at |
+----+---------+--------+------------+
| 1  | 1       | 100    | 2026-01-10 |
| 2  | 1       | 250    | 2026-02-15 |
| 3  | 2       | 80     | 2026-01-20 |
| 4  | 3       | 500    | 2026-03-01 |
| 5  | 5       | 120    | 2026-03-05 |
+----+---------+--------+------------+
```

---

## SELECT, FROM — 가장 기본 형태

```sql
-- All columns
SELECT * FROM users;

-- Specific columns
SELECT id, name FROM users;

-- Expression / computed column
SELECT name, age * 12 AS age_in_months FROM users;

-- Constant
SELECT 'hello' AS greeting, 42 AS answer;
```

`*`는 편하지만 **실무에서는 가급적 컬럼을 명시**하는 게 좋다 — 스키마가 바뀌어도 쿼리가 깨지지 않고, 네트워크/메모리 비용도 줄어든다.

### 별칭 (Alias)

```sql
-- Column alias
SELECT name AS user_name, country AS user_country FROM users;

-- Table alias (very common with JOINs)
SELECT u.name FROM users AS u;

-- AS keyword is optional in many dialects
SELECT u.name FROM users u;
```

---

## WHERE — 행 단위 필터링

조건에 맞는 행만 남긴다. 실행 순서상 `FROM` → `JOIN` 다음, `GROUP BY` 이전에 평가된다.

### 비교 연산자

```sql
SELECT * FROM users WHERE age = 25;
SELECT * FROM users WHERE age <> 25;     -- not equal (also: !=)
SELECT * FROM users WHERE age >= 30;
SELECT * FROM users WHERE age < 30;
```

### 논리 연산자

```sql
-- AND, OR, NOT
SELECT * FROM users WHERE age >= 20 AND country = 'KR';
SELECT * FROM users WHERE country = 'KR' OR country = 'JP';
SELECT * FROM users WHERE NOT country = 'US';
```

연산자 우선순위는 `NOT > AND > OR`. 헷갈리면 괄호로 묶자.

```sql
-- Different meanings!
SELECT * FROM users WHERE country = 'KR' AND age >= 30 OR country = 'JP';
SELECT * FROM users WHERE country = 'KR' AND (age >= 30 OR country = 'JP');
```

### IN — 여러 값 중 하나

```sql
SELECT * FROM users WHERE country IN ('KR', 'JP');

-- Equivalent to:
SELECT * FROM users WHERE country = 'KR' OR country = 'JP';
```

### BETWEEN — 범위

```sql
SELECT * FROM users WHERE age BETWEEN 20 AND 30;
-- Inclusive on both ends: 20 <= age <= 30
```

### LIKE — 패턴 매칭

| 와일드카드 | 의미 |
| --- | --- |
| `%` | 0개 이상의 임의 문자 |
| `_` | 정확히 1개의 임의 문자 |

```sql
SELECT * FROM users WHERE name LIKE 'A%';     -- starts with A
SELECT * FROM users WHERE name LIKE '%e';     -- ends with e
SELECT * FROM users WHERE name LIKE '%li%';   -- contains "li"
SELECT * FROM users WHERE name LIKE '_lice';  -- 5 letters, ends with "lice"

-- Case-insensitive (PostgreSQL)
SELECT * FROM users WHERE name ILIKE 'a%';
```

### NULL 처리 — 매우 중요

`NULL`은 "값이 없음" — `=` 비교가 통하지 않는다.

```sql
-- ❌ Always returns nothing (NULL = NULL is UNKNOWN, not TRUE)
SELECT * FROM users WHERE country = NULL;

-- ✅ Use IS NULL / IS NOT NULL
SELECT * FROM users WHERE country IS NULL;
SELECT * FROM users WHERE country IS NOT NULL;
```

> SQL의 진리값은 **3값 논리 (Three-Valued Logic)** — `TRUE`, `FALSE`, `UNKNOWN`. `NULL`이 들어간 비교는 모두 `UNKNOWN`이고, `WHERE`는 `TRUE`만 통과시키므로 결과에서 제외된다.

`NULL` 대체 함수:

```sql
-- Replace NULL with default
SELECT name, COALESCE(country, 'Unknown') FROM users;

-- Two-argument NULL helper
SELECT NULLIF(age, 0) FROM users;  -- returns NULL if age = 0
```

---

## ORDER BY — 정렬

```sql
SELECT * FROM users ORDER BY age;          -- ASC by default
SELECT * FROM users ORDER BY age DESC;
SELECT * FROM users ORDER BY country ASC, age DESC;  -- multi-key
```

`ORDER BY`는 `SELECT`가 끝난 후 실행되므로 별칭을 쓸 수 있다.

```sql
SELECT name, age * 12 AS months FROM users ORDER BY months DESC;
```

### NULL 정렬 위치

DBMS마다 기본값이 다르다.
- PostgreSQL: `ASC`에서 NULL이 마지막
- MySQL: `ASC`에서 NULL이 처음

명시적으로 지정하려면:

```sql
SELECT * FROM users ORDER BY country NULLS FIRST;
SELECT * FROM users ORDER BY country NULLS LAST;
```

---

## LIMIT / OFFSET — 개수 제한과 건너뛰기

```sql
-- First 10 rows
SELECT * FROM users ORDER BY id LIMIT 10;

-- Skip 20, take 10 (pagination!)
SELECT * FROM users ORDER BY id LIMIT 10 OFFSET 20;
```

DBMS별 문법 차이:

```sql
-- PostgreSQL, MySQL, SQLite
SELECT * FROM users LIMIT 10 OFFSET 20;

-- SQL Server
SELECT TOP 10 * FROM users;
SELECT * FROM users ORDER BY id OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;

-- Oracle 12c+
SELECT * FROM users ORDER BY id OFFSET 20 ROWS FETCH FIRST 10 ROWS ONLY;
```

> `LIMIT`은 반드시 `ORDER BY`와 함께 — 정렬이 없으면 "어떤 10개"인지 보장되지 않는다.

---

## DISTINCT — 중복 제거

```sql
-- Unique countries
SELECT DISTINCT country FROM users;

-- Combination uniqueness
SELECT DISTINCT country, age FROM users;
```

`DISTINCT`는 정렬/해싱 작업이 들어가므로 비싸다. 정말 필요한 경우만 쓰자.

---

## 집계 함수 (Aggregate Functions)

여러 행을 하나의 값으로 압축하는 함수들.

| 함수 | 설명 |
| --- | --- |
| `COUNT(*)` | 전체 행 수 (`NULL` 포함) |
| `COUNT(col)` | `col`이 `NULL`이 아닌 행 수 |
| `COUNT(DISTINCT col)` | `col`의 고유값 개수 |
| `SUM(col)` | 합계 |
| `AVG(col)` | 평균 |
| `MIN(col)` | 최솟값 |
| `MAX(col)` | 최댓값 |

```sql
SELECT
  COUNT(*)        AS total_users,
  COUNT(country)  AS users_with_country,
  AVG(age)        AS avg_age,
  MIN(age)        AS youngest,
  MAX(age)        AS oldest
FROM users;
```

> 집계 함수는 `NULL`을 무시한다 (`COUNT(*)` 제외). 이게 함정이 될 수 있으니 주의.

---

## GROUP BY — 그룹화

지정한 컬럼 값이 같은 행들을 묶어서 그룹별 집계를 만든다.

```sql
SELECT country, COUNT(*) AS user_count, AVG(age) AS avg_age
FROM users
GROUP BY country;
```

결과:
```
+---------+------------+---------+
| country | user_count | avg_age |
+---------+------------+---------+
| KR      | 3          | 27.3    |
| US      | 1          | 30      |
| JP      | 1          | 40      |
+---------+------------+---------+
```

### 중요한 규칙

`SELECT` 절에는 다음 둘 중 하나만 올 수 있다.
1. `GROUP BY`에 있는 컬럼
2. 집계 함수로 감싼 표현식

```sql
-- ❌ name is neither in GROUP BY nor aggregated
SELECT country, name, COUNT(*) FROM users GROUP BY country;

-- ✅
SELECT country, COUNT(*) FROM users GROUP BY country;
```

> MySQL은 기본 설정에서 이 규칙을 어기는 쿼리도 통과시키는데, 이는 표준이 아니며 비결정적인 결과를 낳을 수 있다. `ONLY_FULL_GROUP_BY` 모드를 켜는 게 권장된다.

### 다중 컬럼 그룹화

```sql
SELECT country, age, COUNT(*)
FROM users
GROUP BY country, age;
```

---

## HAVING — 그룹 단위 필터링

`WHERE`는 행 단위 필터, `HAVING`은 그룹 단위 필터.

```sql
-- Countries with more than 1 user
SELECT country, COUNT(*) AS user_count
FROM users
GROUP BY country
HAVING COUNT(*) > 1;
```

### WHERE vs HAVING — 실행 순서로 이해

```
FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
                ↑                    ↑
            row filter          group filter
```

```sql
SELECT country, COUNT(*) AS user_count
FROM users
WHERE age >= 20            -- (3) filter rows BEFORE grouping
GROUP BY country           -- (4) group
HAVING COUNT(*) > 1;       -- (5) filter groups AFTER aggregation
```

> 집계 함수는 `WHERE`에서 못 쓴다. 그룹이 아직 만들어지지 않았으니까.

---

## JOIN — 여러 테이블 결합

여러 테이블을 키 (보통 PK-FK) 로 연결해서 함께 조회한다.

```sql
-- Each user with their orders
SELECT u.name, o.amount, o.created_at
FROM users u
JOIN orders o ON u.id = o.user_id;
```

### JOIN 종류 한눈에

```
       LEFT TABLE          RIGHT TABLE
       ┌─────────┐         ┌─────────┐
       │  A B C  │         │  B C D  │
       └─────────┘         └─────────┘

INNER JOIN        →   B C       (양쪽에 있는 행만)
LEFT  OUTER JOIN  →   A B C     (왼쪽은 모두, 오른쪽 매치 없으면 NULL)
RIGHT OUTER JOIN  →     B C D   (오른쪽은 모두)
FULL  OUTER JOIN  →   A B C D   (양쪽 모두, 매치 없으면 NULL)
CROSS JOIN        →   A×B + ... (조건 없이 모든 조합 — Cartesian Product)
```

### INNER JOIN

양쪽 테이블 모두에 매칭되는 행만 반환.

```sql
SELECT u.name, o.amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id;
-- "INNER" keyword can be omitted; just JOIN
```

위 예제 데이터로는 Dave (주문 없음) 가 결과에서 빠진다.

### LEFT OUTER JOIN

왼쪽 테이블의 모든 행 + 오른쪽에 매치 있으면 그 값, 없으면 `NULL`.

```sql
SELECT u.name, o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
-- Dave appears with amount = NULL
```

**활용**: "주문 없는 사용자 찾기"

```sql
SELECT u.name
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.id IS NULL;
```

### RIGHT OUTER JOIN

`LEFT`의 반대. 실무에서는 `LEFT`로 작성 순서를 바꾸는 경우가 많다.

### FULL OUTER JOIN

양쪽 모두 보존, 매치 없는 쪽은 `NULL`.

```sql
SELECT u.name, o.amount
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;
```

> MySQL은 `FULL OUTER JOIN`을 직접 지원하지 않는다. `LEFT JOIN UNION RIGHT JOIN`으로 우회한다.

### CROSS JOIN — 카르테시안 곱

조건 없이 모든 조합. 결과 행 수 = `M × N`.

```sql
SELECT u.name, c.code
FROM users u
CROSS JOIN countries c;
```

대부분의 경우 의도치 않은 결과지만, "모든 조합 채우기" (예: 모든 (date, product) 조합) 같은 경우엔 유용.

### SELF JOIN — 같은 테이블끼리

직원-매니저 관계처럼 같은 테이블 내에서 관계를 풀 때.

```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

### ON vs USING vs NATURAL JOIN

```sql
-- ON — most explicit, most flexible
SELECT * FROM users u JOIN orders o ON u.id = o.user_id;

-- USING — when join column has the same name on both sides
SELECT * FROM users JOIN orders USING (id);  -- if both have a column named "id"

-- NATURAL JOIN — auto-joins on ALL same-named columns (dangerous!)
SELECT * FROM users NATURAL JOIN orders;
```

> 실무에서는 거의 항상 **`ON`** 을 쓰자. `NATURAL JOIN`은 스키마가 바뀌면 조용히 동작이 변할 수 있어 위험하다.

### JOIN과 WHERE의 결합 — OUTER JOIN 함정

```sql
-- ❌ Looks like LEFT JOIN, but the WHERE filters out NULL rows,
--    effectively turning it into INNER JOIN
SELECT u.name, o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.amount > 100;  -- Dave (NULL amount) gets filtered out

-- ✅ Move the condition into ON to keep LEFT JOIN behavior
SELECT u.name, o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id AND o.amount > 100;
```

`ON`과 `WHERE`의 차이를 정확히 이해해야 한다 — `ON`은 조인 시점에 적용, `WHERE`는 조인 이후에 적용.

---

## 자주 쓰는 패턴

### 그룹별 합계와 개수

```sql
SELECT u.country, COUNT(o.id) AS orders, SUM(o.amount) AS revenue
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.country
ORDER BY revenue DESC NULLS LAST;
```

### 조건부 집계 — `CASE WHEN`

```sql
SELECT
  country,
  COUNT(*)                                          AS total,
  COUNT(CASE WHEN age >= 30 THEN 1 END)             AS adults_30plus,
  SUM(CASE WHEN age >= 30 THEN 1 ELSE 0 END)        AS adults_30plus_alt,
  AVG(CASE WHEN age >= 30 THEN age END)             AS avg_age_30plus
FROM users
GROUP BY country;
```

`CASE WHEN`은 SQL의 `if-else`. 집계와 결합하면 매우 강력하다.

### 결과 합치기 — `UNION` / `UNION ALL`

```sql
-- UNION removes duplicates (slower, requires sort/dedup)
SELECT name FROM korean_users
UNION
SELECT name FROM japanese_users;

-- UNION ALL keeps duplicates (faster, just concatenates)
SELECT name FROM korean_users
UNION ALL
SELECT name FROM japanese_users;
```

> 중복 제거가 정말 필요하지 않으면 **`UNION ALL`** 을 쓰자 — 훨씬 빠르다.

---

## 정리

- `SELECT / FROM / WHERE` 는 행 단위 필터의 기본
- `NULL`은 `=`로 비교 불가 — `IS NULL` 사용
- `GROUP BY`는 `SELECT`에 올 수 있는 컬럼을 제한한다
- `WHERE` (행 필터) 와 `HAVING` (그룹 필터) 의 시점 차이를 정확히
- `JOIN` 종류는 "어느 쪽을 보존할 것인가"로 외우면 쉽다
- `ON` vs `WHERE`, `UNION` vs `UNION ALL` 의 차이는 실무에서 자주 나오는 함정

다음 문서 (`04-sql-advanced-syntax.md`) 에서는 **CTE (`WITH`), 서브쿼리, 윈도우 함수** 같은 한 단계 위의 도구들을 다룬다.
