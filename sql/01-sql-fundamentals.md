# SQL Fundamentals

## SQL이란? (What is SQL?)

**SQL (Structured Query Language)** 은 관계형 데이터베이스 (Relational Database, RDB) 에 데이터를 정의하고 조회·조작하기 위한 표준 언어다.

> "SQL is a domain-specific language used to manage data held in a relational database management system (RDBMS)."

1970년 IBM의 Edgar F. Codd가 발표한 **관계형 모델 (Relational Model)** 논문에서 출발했고, IBM에서 만든 **SEQUEL** 이 현재의 SQL의 시초다.

---

## 선언형 언어 (Declarative Language)

SQL의 가장 큰 특징은 **선언형 (Declarative)** 이라는 점이다.

| 구분 | Imperative (명령형) | Declarative (선언형) |
| --- | --- | --- |
| 표현 | "어떻게 (How)" | "무엇을 (What)" |
| 예시 언어 | C, Java, JavaScript (절차적 부분) | SQL, HTML, Prolog |
| 작성 방식 | 단계별 절차를 명시 | 원하는 결과만 기술 |

### 예시 비교

같은 작업을 명령형과 선언형으로 비교하면:

```javascript
// Imperative: tell the machine HOW to do it
const result = [];
for (const user of users) {
  if (user.age >= 20) {
    result.push(user.name);
  }
}
```

```sql
-- Declarative: tell the machine WHAT you want
SELECT name
FROM users
WHERE age >= 20;
```

> SQL을 작성하는 사람은 **결과의 모습**만 기술하고, **실제 어떻게 가져올지 (실행 계획)** 는 데이터베이스 옵티마이저 (Optimizer) 가 결정한다. 이 점이 다음 문서 `02-sql-execution-flow.md`의 핵심 주제로 이어진다.

---

## SQL 표준과 방언 (Standard & Dialects)

SQL은 ANSI/ISO 표준이 존재한다.

| 표준 | 연도 | 주요 추가 기능 |
| --- | --- | --- |
| SQL-86 / SQL-89 | 1986/1989 | 최초 표준 |
| SQL-92 | 1992 | 가장 많이 인용되는 기준 |
| SQL:1999 | 1999 | CTE (`WITH`), 정규식, 트리거 |
| SQL:2003 | 2003 | 윈도우 함수 (`OVER`), `MERGE`, XML |
| SQL:2008 | 2008 | `TRUNCATE`, `INSTEAD OF` 트리거 |
| SQL:2011 | 2011 | 시간 데이터 (Temporal Tables) |
| SQL:2016 | 2016 | JSON 지원, 행 패턴 매칭 |

**하지만 표준을 100% 따르는 DBMS는 없다.** 각 벤더는 자체 확장 (방언, Dialect) 을 추가한다.

| DBMS | 방언 이름 | 특징 |
| --- | --- | --- |
| Oracle | PL/SQL | 강력한 절차적 확장 |
| Microsoft SQL Server | T-SQL (Transact-SQL) | 변수, 흐름 제어 |
| PostgreSQL | PL/pgSQL | 함수 정의 가능 |
| MySQL | (자체 방언) | `LIMIT`, `AUTO_INCREMENT` |
| SQLite | (경량 방언) | 일부 표준 미지원 |

> 그래서 같은 "상위 N개 행 가져오기"도 DBMS마다 문법이 다르다:
> - PostgreSQL/MySQL: `SELECT * FROM users LIMIT 10;`
> - SQL Server: `SELECT TOP 10 * FROM users;`
> - Oracle (12c+): `SELECT * FROM users FETCH FIRST 10 ROWS ONLY;`

---

## SQL 명령어 분류 (SQL Command Categories)

SQL 명령어는 목적에 따라 4가지로 분류한다.

### 1. DDL (Data Definition Language) — 데이터 정의어

테이블, 인덱스, 뷰 등 **스키마 (Schema)** 를 정의·수정·삭제한다.

```sql
-- Create a table
CREATE TABLE users (
  id        BIGINT PRIMARY KEY,
  email     VARCHAR(255) UNIQUE NOT NULL,
  age       INT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Modify schema
ALTER TABLE users ADD COLUMN nickname VARCHAR(50);

-- Drop schema
DROP TABLE users;

-- Truncate (delete all rows, keep structure)
TRUNCATE TABLE users;
```

대표 키워드: `CREATE`, `ALTER`, `DROP`, `TRUNCATE`, `RENAME`

### 2. DML (Data Manipulation Language) — 데이터 조작어

테이블 안의 **데이터 (Row)** 를 조회·삽입·수정·삭제한다.

```sql
-- SELECT (read)
SELECT id, email FROM users WHERE age >= 20;

-- INSERT (create)
INSERT INTO users (id, email, age) VALUES (1, 'a@b.com', 25);

-- UPDATE (update)
UPDATE users SET age = 26 WHERE id = 1;

-- DELETE (delete)
DELETE FROM users WHERE id = 1;
```

대표 키워드: `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `MERGE`

> 일부 분류에서는 `SELECT`만 따로 떼어 **DQL (Data Query Language)** 이라고 부르기도 한다.

### 3. DCL (Data Control Language) — 데이터 제어어

권한 (Privilege) 을 부여하거나 회수한다.

```sql
-- Grant SELECT permission to a user
GRANT SELECT ON users TO analyst_user;

-- Revoke permission
REVOKE SELECT ON users FROM analyst_user;
```

대표 키워드: `GRANT`, `REVOKE`

### 4. TCL (Transaction Control Language) — 트랜잭션 제어어

**트랜잭션 (Transaction)** — 여러 개의 DML 작업을 하나의 단위로 묶는 — 을 제어한다.

```sql
BEGIN;                    -- start transaction
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;                   -- save changes

-- Or rollback if something goes wrong
ROLLBACK;

-- Save intermediate state
SAVEPOINT before_update;
ROLLBACK TO SAVEPOINT before_update;
```

대표 키워드: `BEGIN` / `START TRANSACTION`, `COMMIT`, `ROLLBACK`, `SAVEPOINT`

> 트랜잭션은 **ACID** 속성 — Atomicity (원자성), Consistency (일관성), Isolation (고립성), Durability (지속성) — 을 보장한다.

---

## 관계형 모델 (Relational Model)

SQL이 다루는 데이터 구조의 기본 단위.

```
Database
└── Schema
    └── Table (= Relation)
        ├── Column (= Attribute)
        └── Row (= Tuple)
```

### 핵심 개념

- **Table (테이블)**: 행 (Row) 과 열 (Column) 로 구성된 2차원 구조
- **Primary Key (기본 키, PK)**: 각 행을 유일하게 식별하는 컬럼
- **Foreign Key (외래 키, FK)**: 다른 테이블의 PK를 참조하는 컬럼 — 테이블 간 **관계 (Relationship)** 를 만듦
- **Unique Key**: 중복을 허용하지 않는 컬럼
- **Index (인덱스)**: 검색 속도를 높이기 위한 자료구조 (대부분 B-Tree)

### 예시: 1:N 관계

```sql
CREATE TABLE users (
  id    BIGINT PRIMARY KEY,
  name  VARCHAR(50)
);

CREATE TABLE orders (
  id       BIGINT PRIMARY KEY,
  user_id  BIGINT NOT NULL,
  amount   DECIMAL(10, 2),
  FOREIGN KEY (user_id) REFERENCES users(id)
);
```

`users` 1명 → `orders` N개 (1:N 관계). `JOIN`으로 두 테이블을 함께 조회할 수 있다.

---

## SQL은 "집합 (Set) 기반 언어"다

SQL을 이해할 때 가장 중요한 사고 전환 중 하나는 **"한 번에 한 행씩 처리"가 아니라 "집합 단위로 처리"** 한다는 점이다.

```sql
-- This doesn't loop row-by-row in your head;
-- it operates on the entire set "users".
UPDATE users SET age = age + 1 WHERE country = 'KR';
```

이는 SQL이 **수학의 집합론 (Set Theory)** 과 **관계 대수 (Relational Algebra)** 에 기반하기 때문이다. `JOIN`, `UNION`, `INTERSECT`, `EXCEPT` 모두 집합 연산이다.

---

## 정리

- SQL은 **선언형 언어**로, 결과의 모습만 기술하면 옵티마이저가 실행 방법을 결정한다
- ANSI/ISO **표준**이 있지만 벤더별 **방언**이 존재한다
- 명령어는 목적별로 **DDL / DML / DCL / TCL** 4가지로 분류
- 데이터 모델의 핵심은 **테이블, 키, 관계**
- SQL은 **집합 기반 (Set-based)** 사고를 요구한다

다음 문서에서는 우리가 작성한 SQL이 **어떻게 파싱되고, 최적화되고, 실행되는지** 를 살펴본다.
