# Query Engines

지금까지 SQL 문법과 실행 흐름을 봤다. 이제 그 SQL을 **실제로 실행하는 시스템** 의 세계로 들어가보자. "데이터베이스" 라고 한 마디로 묶기엔 종류가 너무 많고, 각자 풀려는 문제가 다르다.

---

## DBMS와 Query Engine의 차이

먼저 큰 그림.

```
┌─────────────────────────────────────────────────────┐
│             전통적 DBMS (Monolithic)                 │
│  ┌─────────────────────────────────────────────┐    │
│  │            Query Engine                     │    │
│  └─────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────┐    │
│  │            Storage Engine                   │    │
│  │   (자체 디스크 포맷, 파일을 직접 관리)        │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
            예: MySQL, PostgreSQL, Oracle


┌─────────────────────────────────────────────────────┐
│       분산 Query Engine (Storage-Compute 분리)       │
│  ┌─────────────────────────────────────────────┐    │
│  │            Query Engine                     │    │
│  │  (Trino, Spark SQL, Presto, ...)            │    │
│  └────────────┬──────────────┬─────────────────┘    │
│               │              │                      │
└───────────────┼──────────────┼──────────────────────┘
                ▼              ▼
        ┌──────────────┐  ┌──────────────┐
        │   S3 (Parquet)│  │    Hive     │
        │   (Iceberg)   │  │    Kafka    │
        │   ...         │  │    MySQL    │
        └──────────────┘  └──────────────┘
              외부 저장소 (Connector로 연결)
```

| 구분 | 전통 DBMS | 분산 쿼리 엔진 |
| --- | --- | --- |
| 저장소 | 자체 (Coupled) | 외부 (Decoupled) |
| 데이터 형식 | DB 전용 포맷 | Parquet, ORC, Avro 등 표준 포맷 |
| 트랜잭션 | ACID 보장 | 보통 read-only / 제한적 |
| 강점 | 트랜잭션, 일관성 | 대규모 분석, 다양한 소스 통합 |
| 사용 사례 | OLTP | OLAP / Data Lake |

---

## OLTP vs OLAP — 워크로드의 분류

쿼리 엔진을 이해하려면 먼저 **어떤 종류의 작업을 하는지** 를 구분해야 한다.

| 항목 | OLTP (Online Transaction Processing) | OLAP (Online Analytical Processing) |
| --- | --- | --- |
| 주 작업 | 짧은 읽기/쓰기 트랜잭션 | 큰 범위 집계/분석 |
| 쿼리 크기 | 1~수십 행 | 수백만~수십억 행 |
| 응답 시간 | 밀리초 | 초~분 |
| 사용자 | 애플리케이션 (수많은 동시 요청) | 분석가/대시보드 |
| 인덱스 | B-Tree (행 단위 접근) | 컬럼 압축 (스캔 최적화) |
| 저장 방식 | **Row-oriented** | **Column-oriented** |
| 예시 | "주문 1건 결제 처리" | "지난 1년 국가별 매출 추이" |
| 대표 시스템 | MySQL, PostgreSQL, Oracle | ClickHouse, BigQuery, Trino |

### Row vs Column Storage — 왜 OLAP은 컬럼 기반인가?

```
Row-oriented (OLTP에 유리):
[id=1, name=Alice, country=KR, age=25]
[id=2, name=Bob,   country=US, age=30]
[id=3, name=Carol, country=KR, age=22]
디스크 배치: 1,Alice,KR,25 | 2,Bob,US,30 | 3,Carol,KR,22

Column-oriented (OLAP에 유리):
ids:       [1, 2, 3, ...]
names:     [Alice, Bob, Carol, ...]
countries: [KR, US, KR, ...]
ages:      [25, 30, 22, ...]
```

`SELECT AVG(age) FROM users` 를 1억 행 테이블에서 실행한다고 생각해보자.
- Row-oriented: 모든 컬럼을 읽으면서 age만 추출 → 디스크 I/O 낭비
- Column-oriented: age 컬럼만 읽으면 끝 → 압축률도 훨씬 좋음 (같은 타입이 연속)

---

## 쿼리 엔진의 카테고리

### 1. 전통 RDBMS (OLTP 중심)

| 시스템 | 특징 |
| --- | --- |
| **MySQL** | 가장 널리 쓰이는 오픈소스 RDBMS, 빠른 읽기 |
| **PostgreSQL** | 표준 준수, 풍부한 기능 (JSON, GIS, 윈도우 함수 등) |
| **Oracle DB** | 엔터프라이즈 기능, PL/SQL |
| **SQL Server** | Microsoft 생태계, T-SQL |
| **SQLite** | 임베디드 (앱 내장), 단일 파일 |

이들은 보통 **단일 노드** 또는 **마스터-레플리카** 구성. 수평 확장 (Sharding) 은 추가 작업이 필요.

### 2. NewSQL — OLTP의 분산 버전

ACID 트랜잭션을 유지하면서 수평 확장하려는 시도.

| 시스템 | 특징 |
| --- | --- |
| **CockroachDB** | PostgreSQL 호환, 글로벌 분산 |
| **Google Spanner** | 외부 일관성, TrueTime |
| **TiDB** | MySQL 호환, HTAP (OLTP + OLAP 둘 다) |
| **YugabyteDB** | PostgreSQL 호환, 분산 |

### 3. OLAP 전용 DB (Real-time Analytics)

| 시스템 | 특징 |
| --- | --- |
| **ClickHouse** | 초고속 컬럼 DB, 단일 테이블 집계에 압도적 |
| **Apache Druid** | 실시간 OLAP, 시계열에 강함 |
| **Apache Pinot** | LinkedIn 출신, 실시간 분석 |
| **Apache Doris** | MPP, MySQL 프로토콜 호환 |

> 이들은 **자체 저장소**를 가지면서 컬럼 기반으로 OLAP에 특화됐다.

### 4. 분산 SQL 쿼리 엔진 (Storage-Decoupled)

자체 저장소 없이, **다양한 외부 데이터 소스를 SQL로 통합 조회**.

| 시스템 | 출신 | 특징 |
| --- | --- | --- |
| **Trino** | Facebook → Trino Software Foundation | MPP, 인터랙티브 쿼리 (초~분) |
| **Presto (PrestoDB)** | Facebook → Linux Foundation | Trino의 형제, 갈라짐 |
| **Apache Spark SQL** | UC Berkeley AMPLab | 범용 분산 처리, 배치/스트림 |
| **Apache Hive** | Facebook (2008) | MapReduce 위 SQL, 배치 |
| **Apache Impala** | Cloudera | HDFS 위 인터랙티브 |
| **Apache Drill** | MapR | Schema-free 조회 |

### 5. 클라우드 데이터 웨어하우스 (DWH)

위의 4번을 매니지드 형태로 제공 + 자체 저장소.

| 시스템 | 제공 | 특징 |
| --- | --- | --- |
| **Google BigQuery** | GCP | Serverless, Dremel 기반 |
| **Snowflake** | 멀티 클라우드 | Storage/Compute 완전 분리, 가상 웨어하우스 |
| **Amazon Redshift** | AWS | PostgreSQL 기반 시작, 컬럼 스토리지 |
| **Databricks SQL** | 멀티 클라우드 | Spark + Delta Lake |

---

## Trino 깊이 보기

### 등장 배경

```
2008  Facebook이 Hive 사용 시작 (HDFS + MapReduce)
       → 배치는 OK, 인터랙티브 분석은 너무 느림 (분~시간)
2012  Facebook이 Presto 개발 시작
       → 메모리 기반 MPP로 인터랙티브 쿼리 (초~분)
2013  Presto 오픈소스 공개
2018  창립자들 (Facebook 출신)이 PrestoSQL로 fork
2020  PrestoSQL → Trino로 리브랜딩
```

오늘날 두 갈래:
- **PrestoDB** (Linux Foundation, Facebook이 주도)
- **Trino** (Trino Software Foundation, 원작자 그룹)

대부분의 "최신" 기능과 커뮤니티 활동은 **Trino** 쪽이 활발하다.

### 핵심 컨셉: SQL on Anything

> "Trino is a query engine. It does **not** store data. It **federates queries** across many data sources using SQL."

```
        ┌─────────────────────────┐
        │      SQL Query          │
        │ "join Hive table with   │
        │  MySQL table, filter,   │
        │  aggregate"             │
        └────────────┬────────────┘
                     │
                     ▼
        ┌─────────────────────────┐
        │         Trino           │
        │  ┌──────────────────┐   │
        │  │  Query Planner   │   │
        │  │  Optimizer       │   │
        │  │  Distributed     │   │
        │  │  Execution       │   │
        │  └──────────────────┘   │
        └────┬─────┬─────┬────────┘
             │     │     │
   ┌─────────┘     │     └─────────┐
   ▼               ▼               ▼
┌──────┐      ┌────────┐     ┌──────────┐
│ Hive │      │ MySQL  │     │ Kafka    │
│ S3   │      │ Postgres     │ Iceberg  │
└──────┘      └────────┘     └──────────┘
```

### 아키텍처

```
                  ┌─────────────────┐
                  │   Coordinator   │
                  │  - Parsing      │
                  │  - Planning     │
                  │  - Scheduling   │
                  └────────┬────────┘
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
       ┌─────────┐    ┌─────────┐    ┌─────────┐
       │ Worker  │    │ Worker  │    │ Worker  │
       │  Node   │    │  Node   │    │  Node   │
       └────┬────┘    └────┬────┘    └────┬────┘
            ▼              ▼              ▼
       (data source connectors read in parallel)
```

- **Coordinator**: 클라이언트 요청 받음, 쿼리 파싱/최적화/실행 계획 분배
- **Workers**: 실제 데이터 처리 (병렬)
- **Connector**: 데이터 소스별 어댑터 (Hive, Iceberg, MySQL, Postgres, Kafka, Cassandra, MongoDB, Elasticsearch ...)

### Trino의 강점

#### (1) MPP — Massively Parallel Processing

쿼리를 잘게 쪼개 (Stage → Task → Split) 수십~수백 노드에서 병렬 실행. 모든 처리가 메모리 위주 → Hive 대비 수십 배 빠름.

#### (2) Federation — 여러 소스 한 번에

```sql
-- Join data from S3 (Hive) and MySQL in one query
SELECT u.name, h.total_amount
FROM mysql.crm.users u
JOIN hive.analytics.daily_orders h ON u.id = h.user_id
WHERE h.dt = DATE '2026-04-01';
```

`catalog.schema.table` 형식으로 다른 소스를 자유롭게 섞는다.

#### (3) ANSI SQL 호환성

윈도우 함수, CTE, 복잡한 JOIN 등 표준 SQL 기능을 폭넓게 지원.

### Trino의 약점

- 자체 저장소가 없으므로 **인덱스가 없다** — 풀 스캔 위주
- **트랜잭션 없음** (read-heavy 분석용)
- 메모리 기반이라 **메모리 부족 시 쿼리 실패** (Spark는 디스크로 spill)
- **장기 실행 (시간 단위) 배치엔 부적합** — 그건 Spark가 더 안정적

---

## Trino vs Spark SQL vs Hive — 언제 무엇을 쓸까

| 항목 | Hive | Spark SQL | Trino |
| --- | --- | --- | --- |
| 등장 | 2008 | 2014 | 2012 (Presto) → 2020 (Trino) |
| 실행 모델 | MapReduce / Tez | RDD / DataFrame | MPP (in-memory) |
| 응답 시간 | 분~시간 (배치) | 초~분 | 초~분 |
| 장기 작업 안정성 | 매우 좋음 | 좋음 (디스크 spill) | 약함 (메모리 부족 시 fail) |
| 인터랙티브 쿼리 | 부적합 | 보통 | **최적** |
| ETL/배치 | 적합 (구식) | **최적** | 부적합 |
| 머신러닝 | 없음 | **MLlib 내장** | 없음 |
| 스트리밍 | 없음 | **Structured Streaming** | 없음 |
| 다중 소스 조회 | 제한적 | 가능 | **최적 (수십 가지 connector)** |

> 실무에선 **함께** 쓰는 경우가 많다 — 야간 ETL은 Spark, 주간 대시보드 / 분석가 ad-hoc 쿼리는 Trino.

---

## 클라우드 DWH 한 줄 요약

| 시스템 | 한 줄 |
| --- | --- |
| **BigQuery** | 서버리스 — 클러스터 관리 불필요, 바이트 스캔 단위 과금. Google Dremel 기반 |
| **Snowflake** | Storage/Compute 완전 분리, 가상 웨어하우스 단위로 컴퓨트를 켜고 끔. 멀티 클라우드 |
| **Redshift** | AWS 네이티브, PostgreSQL 호환, 노드 클러스터 직접 관리 (RA3 인스턴스로 일부 분리) |
| **Databricks SQL** | Spark + Delta Lake 위에 SQL 엔드포인트, ML과 통합 |

---

## 선택 기준 — 어떤 엔진을 쓸까?

### 결정 트리 (간단 버전)

```
앱이 트랜잭션 위주? → MySQL / PostgreSQL (또는 NewSQL)
                       │
                  분석 중심? ─────┐
                                 │
        데이터가 한 시스템(파일)에? ── ClickHouse / Druid (자체 저장 OLAP)
                                 │
        여러 소스를 SQL로 통합? ── Trino / Presto
                                 │
        대규모 ETL / ML / 스트림? ── Spark
                                 │
        매니지드 + 자동 확장 원함? ── BigQuery / Snowflake / Redshift
```

### 더 구체적인 기준

- **데이터 크기**: GB → 단일 노드 RDBMS, TB → 컬럼형 OLAP, PB → 분산 쿼리 엔진 / 클라우드 DWH
- **쿼리 응답 요구**: ms → OLTP DB, 초 → ClickHouse / Trino, 분 → Spark
- **워크로드**: 트랜잭션 → OLTP, 분석 ad-hoc → Trino, 정기 ETL → Spark, 대시보드 → DWH
- **운영 부담**: 직접 운영 → 오픈소스 Trino/Spark, 매니지드 원함 → 클라우드 DWH
- **데이터 소스 다양성**: 한 곳에 모음 → 자체 저장 DB, 여러 소스 → 페더레이션 (Trino)

---

## 정리

- **DBMS**는 보통 저장소 + 쿼리 엔진이 결합된 형태. **분산 쿼리 엔진**은 저장소와 분리되어 외부 데이터를 SQL로 조회한다
- **OLTP** (트랜잭션) vs **OLAP** (분석) 의 구분이 모든 선택의 출발점
- 컬럼 기반 저장 (Columnar) 이 OLAP에 압도적으로 유리한 이유는 "필요한 컬럼만 읽고 압축률도 좋아서"
- **Trino**는 인터랙티브 분석 + 다중 데이터 소스 페더레이션의 사실상 표준
- **Spark**는 ETL/ML/스트리밍의 범용, **Hive**는 배치 안정성, **Trino**는 인터랙티브 — 셋은 보완적
- 클라우드 시대엔 **BigQuery / Snowflake / Redshift** 가 매니지드 옵션으로 자리 잡음

---

## 더 깊이 보고 싶다면

다음 주제로 확장 가능:
- 컬럼 포맷 자체: **Parquet**, **ORC**, **Avro** 의 구조와 차이
- 테이블 포맷: **Apache Iceberg**, **Delta Lake**, **Apache Hudi** — Data Lakehouse의 핵심
- Trino 쿼리 최적화: Cost-Based Optimizer, Dynamic Filtering, Join Reordering
- 데이터 모델링: **Star Schema**, **Snowflake Schema**, **Data Vault**
