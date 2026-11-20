---
title:  "[DE/용어] JDBC 읽기 파티션 (predicates / partitionColumn)"
excerpt: "Spark가 Oracle 같은 DB를 병렬로 읽는 단위를 정하는 두 가지 방법과 함정"

categories:
  - DataEngineering
tags:
  - [Glossary, Spark, JDBC, Oracle, Parallel Read]

toc: true
toc_sticky: true
published: true

date: 2026-11-19
last_modified_at: 2026-11-19
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-19)    
[Spark JDBC data source](https://spark.apache.org/docs/latest/sql-data-sources-jdbc.html)    
[PySpark DataFrameReader.jdbc](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrameReader.jdbc.html)
{: .notice--warning}

# 0. 이 글을 읽기 전에

"월별로 나눠서 병렬로 조회한다"고 할 때, Spark 안에서는 정확히 무슨 일이 일어나는지 설명합니다. 이 글의 "파티션"은 **읽는 단위**이고, Delta 테이블의 **저장 파티셔닝**(→ Delta 테이블 파티셔닝 글)과는 다른 개념입니다.

# 1. 한 줄 정의

Spark가 JDBC로 DB를 읽을 때, 결과를 **여러 조각(파티션)으로 나눠서 여러 워커가 동시에 읽는 방식**과 그 조각의 기준입니다.

# 2. 왜 필요한가: 아무 설정이 없으면?

```python
df = spark.read.jdbc(url=jdbc_url, table="SALES.ORDERS", properties=conn_props)
```

파티션 관련 옵션을 하나도 주지 않으면 Spark는 **하나의 파티션, 하나의 연결**로 테이블 전체를 읽습니다. 데이터가 크면 **한 개의 task**가 모든 것을 읽으므로 느립니다. 읽기를 **여러 조각으로 나누면 여러 task가 각자 연결을 열어 동시에** 읽습니다.

```
[분할 없음]   task 1 ───────────────────────────────→ 전체 1억 행 (느림)

[분할 4개]    task 1 ──→ 2,500만 행
              task 2 ──→ 2,500만 행      동시에 진행 (빠름, 단 DB 연결 4개)
              task 3 ──→ 2,500만 행
              task 4 ──→ 2,500만 행
```

# 3. 분할 방법 두 가지

## 3-1. 방법 A — `predicates`: 조각의 조건을 직접 지정

```python
predicates = [
    "order_dt >= DATE '2026-01-01' AND order_dt < DATE '2026-02-01'",
    "order_dt >= DATE '2026-02-01' AND order_dt < DATE '2026-03-01'",
    "order_dt >= DATE '2026-03-01' AND order_dt < DATE '2026-04-01'",
]
df = spark.read.jdbc(
    url=jdbc_url, table="SALES.ORDERS",
    predicates=predicates, properties=conn_props,
)
```

- 리스트의 **식 하나가 DataFrame 파티션 하나**를 정의합니다. (PySpark API: "각각 WHERE 절에 넣을 수 있는 식이며, 하나가 DataFrame의 파티션 하나를 정의")
- Spark는 각 파티션에서 대략 다음과 같은 쿼리를 Oracle로 보냅니다. (개념 설명)

```sql
SELECT ... FROM SALES.ORDERS WHERE order_dt >= DATE '2026-01-01' AND order_dt < DATE '2026-02-01'
SELECT ... FROM SALES.ORDERS WHERE order_dt >= DATE '2026-02-01' AND order_dt < DATE '2026-03-01'
...
```

### ⚠️ 조건을 직접 만들 때의 책임

Spark는 predicate들이 **서로 겹치지 않는지, 전체를 빠짐없이 덮는지 검사하지 않습니다.**

| 실수 | 결과 |
|---|---|
| 두 predicate가 **겹침** (예: `<= '2026-01-31'`과 `>= '2026-01-31'`) | 겹친 행이 **중복**되어 읽힘 |
| 어떤 구간이 **빠짐** | 그 구간의 행이 **누락** |
| 컬럼에 **NULL**이 있는 행 | 어떤 predicate에도 안 걸려 **누락** |

> 안전한 패턴: 구간을 **반열린 구간 `[시작, 다음 시작)`** 으로 만들고(`>= 시작 AND < 다음시작`), 날짜가 NULL일 수 있다면 `IS NULL` 조건을 별도 predicate로 추가합니다.
{: .notice--info}

## 3-2. 방법 B — `partitionColumn` + `lowerBound` + `upperBound` + `numPartitions`

```python
df = (spark.read.format("jdbc")
      .option("url", jdbc_url)
      .option("dbtable", "SALES.ORDERS")
      .option("partitionColumn", "order_seq")     # 숫자/날짜/타임스탬프 컬럼
      .option("lowerBound", "1")
      .option("upperBound", "1000")
      .option("numPartitions", "4")
      .load())
```

Spark가 **구간을 자동으로 계산**합니다. 위 예에서는 `(1000 − 1) / 4 ≈ 250`씩 끊어 대략 이런 조건이 만들어집니다. (개념 설명이며 세부 조건식은 Spark 내부 구현에 따릅니다)

```
파티션 1:  order_seq < 251      (NULL 포함)
파티션 2:  order_seq >= 251 AND order_seq < 501
파티션 3:  order_seq >= 501 AND order_seq < 751
파티션 4:  order_seq >= 751
```

- 네 옵션(`partitionColumn`, `lowerBound`, `upperBound`, `numPartitions`)은 **모두 함께** 지정해야 합니다.
- `partitionColumn`은 **numeric, date, timestamp** 컬럼이어야 합니다.
- ⚠️ **`lowerBound`/`upperBound`는 행을 거르는 필터가 아닙니다.** 파티션 **간격(stride)을 정하는 용도**일 뿐이고 **테이블의 모든 행이 파티션되어 반환됩니다.** 범위를 벗어난 값은 첫/마지막 파티션에 포함됩니다.
- **`query` 옵션과 `partitionColumn`은 함께 쓸 수 없습니다.** 서브쿼리를 쓰고 싶다면 `dbtable`에 서브쿼리를 넣으세요.

```python
.option("dbtable", "(SELECT order_dt, order_id, amount FROM SALES.ORDERS WHERE status_cd = 'OK') src")
```

- `column`(partitionColumn)과 `predicates`를 **둘 다** 주면 `column` 쪽이 사용됩니다. (PySpark API)

## 3-3. 어느 쪽을 쓸까

| 상황 | 추천 |
|---|---|
| 숫자 PK(시퀀스)가 **골고루 분포** | 방법 B (자동 분할) |
| 월·일 같은 **비즈니스 구간**으로 나누고 싶음 | 방법 A (predicates) |
| 구간별로 **개별 실패/재처리**가 필요 | 방법 A로 정의하되, **쓰기까지 청크 단위로 분리** (→ 청크 글) |
| 데이터가 한쪽에 **쏠려** 있음 | 방법 B는 일부 파티션만 커질 수 있음 → predicates로 직접 균형 |

# 4. 파티션 수 = 동시 연결 수

`numPartitions`는 **읽기·쓰기 병렬성의 최대 파티션 수**이며, 공식 문서는 "이는 **동시에 열리는 JDBC 연결의 최대 수**도 결정한다"고 설명합니다. 쓸 때 파티션 수가 이 한도를 넘으면 `coalesce(numPartitions)`로 줄입니다.

- Oracle의 **허용 세션 수**를 넘지 않게 정해야 합니다.
- predicates를 쓸 때도 predicate 개수만큼 파티션이 생기고 그만큼 연결이 열릴 수 있습니다. 개수가 매우 많다면 병렬도 제어(동시 실행 청크 수 제한)를 고려하세요.

# 5. `fetchsize`: 한 번에 가져오는 행 수

```python
connection_properties = {"user": "...", "password": "...", "fetchsize": "10000"}
```

- `fetchsize`는 **DB 왕복 한 번에 가져올 행 수**입니다. 공식 문서: 기본값이 낮은 드라이버(**예: Oracle은 10행**)에서 성능에 도움이 됩니다.
- 너무 크게 잡으면 워커 메모리 사용이 늘 수 있으니 컬럼 폭과 행 수를 보고 조정하세요.

# 6. 가장 중요한 연결고리: "읽기 파티션 N개 = 쓰기 커밋은 1번"

```python
df = spark.read.jdbc(..., predicates=[월1, 월2, ..., 월12], ...)   # 읽기 파티션 12개
df.write.format("delta").mode("append").saveAsTable("main.l0.orders")
```

- 읽기 파티션이 12개여도 **DataFrame은 하나**이고, 쓰기는 **Spark job 하나 → Delta 커밋 하나**입니다.
- 그래서 12개 중 한 파티션(task)이 끝내 실패하면 **쓰기 전체가 실패**합니다. (→ Delta 트랜잭션 글, `spark.task.maxFailures` 글)
- 월별로 **개별 성공/실패**가 필요하다면, 파티션 하나당 `읽기 + 쓰기`를 **따로** 수행하는 **청크** 방식으로 나눕니다.

# 7. 직접 확인해 보기

```python
print(df.rdd.getNumPartitions())     # 읽기 파티션 수 (predicates 개수와 같아야 함)
```

- Spark UI의 해당 스테이지에서 **task 수**와 task별 처리 행 수·시간을 보면, 파티션이 고르게 분배됐는지 알 수 있습니다. 한 task만 오래 걸리면 **데이터 쏠림**입니다.

# 8. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "`lowerBound`/`upperBound`로 범위를 필터한다" | **필터가 아님.** stride 계산용이고 모든 행이 반환됨 |
| "파티션 수를 늘릴수록 빨라진다" | DB 연결/부하가 한계. `numPartitions`가 **최대 동시 연결 수** |
| "predicates에서 겹침/누락은 Spark가 알려준다" | 검사하지 않음. **직접 상호배타·전체포괄**로 만들어야 함 |
| "읽기 파티션이 나뉘었으니 쓰기도 나뉜다" | 쓰기는 **한 번(커밋 1회)**. 부분 성공을 원하면 청크 단위로 따로 쓴다 |
| "`query`와 `partitionColumn`을 같이 쓰면 된다" | **같이 쓸 수 없음**. 서브쿼리는 `dbtable`로 |

# 9. 한 줄 정리

- JDBC 읽기 파티션 = Spark가 DB를 **병렬로 읽는 단위**. 파티션 하나 = task 하나 = DB 연결 하나.
- 방법은 **`predicates`(조건을 직접)** 또는 **`partitionColumn`+bounds+`numPartitions`(자동 분할)**.
- bounds는 필터가 아니고, predicates는 **겹침/누락을 직접 책임**져야 한다.
- 파티션 수는 **동시 연결 수**와 같으니 Oracle 한도를 고려한다. `fetchsize`로 왕복 횟수를 줄인다.
- 읽기 파티션이 여러 개여도 **쓰기 커밋은 한 번**이라는 점이 "한 달 실패 = 전체 실패"의 뿌리다.

# 관련 용어
청크(Chunk), Delta 트랜잭션(ACID), Delta 테이블 파티셔닝, spark.task.maxFailures

## 🔗 참고
- [Spark JDBC data source](https://spark.apache.org/docs/latest/sql-data-sources-jdbc.html)
- [PySpark DataFrameReader.jdbc](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrameReader.jdbc.html)
