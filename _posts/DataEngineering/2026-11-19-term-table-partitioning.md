---
title:  "[DE/용어] Delta 테이블 파티셔닝(Partitioning)"
excerpt: "테이블 파일을 컬럼 값별로 나눠 저장하는 방식, 그리고 지금은 왜 '가급적 하지 말라'고 하는지"

categories:
  - DataEngineering
tags:
  - [Glossary, Delta Lake, Partitioning, Data Layout]

toc: true
toc_sticky: true
published: true

date: 2026-11-19
last_modified_at: 2026-11-19
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-19)    
[When to partition tables on Databricks](https://docs.databricks.com/aws/en/tables/partitions)    
[Use liquid clustering for tables](https://docs.databricks.com/aws/en/delta/clustering)    
[Row-level concurrency](https://docs.databricks.com/aws/en/optimizations/isolation/row-level-concurrency)
{: .notice--warning}

# 0. 이 글을 읽기 전에

"파티셔닝"이라는 단어는 데이터 엔지니어링에서 **서로 다른 세 가지 의미**로 쓰입니다. 먼저 구분하겠습니다.

| 용어 | 의미 | 이 시리즈의 글 |
|---|---|---|
| **Delta 테이블 파티셔닝** (이 글) | 테이블의 **파일을 저장하는 방식**. `PARTITIONED BY` | 이 글 |
| **JDBC 읽기 파티션** | Spark가 Oracle 같은 DB를 **병렬로 읽는 단위** | JDBC 읽기 파티션 글 |
| **Spark(DataFrame) 파티션** | Spark가 메모리에서 데이터를 **나눠 처리하는 단위** | (별도) |

월별로 조회하는 것은 두 번째 의미이고, 이 글은 첫 번째 의미를 다룹니다.

# 1. 한 줄 정의

**테이블의 데이터를 특정 컬럼의 값별로 나눠서 저장**하는 방식입니다. 해당 컬럼으로 조회하면 **필요한 파티션만 읽을 수 있습니다.** 선언은 `PARTITIONED BY`로 합니다.

# 2. 어떻게 저장되는가

`order_ym`(주문 연월) 컬럼으로 파티셔닝한 테이블을 개념적으로 그리면 다음과 같습니다.

```
orders/
 ├─ order_ym=2026-01/     ← 1월 데이터 파일들
 │    ├─ part-0001.parquet
 │    └─ part-0002.parquet
 ├─ order_ym=2026-02/     ← 2월 데이터 파일들
 │    └─ part-0003.parquet
 └─ order_ym=2026-03/
      └─ part-0004.parquet
```

```sql
SELECT SUM(amount) FROM main.l0.orders WHERE order_ym = '2026-02';
-- → order_ym=2026-02 폴더만 읽는다 (나머지 폴더는 건너뜀: partition pruning)
```

> 위 폴더 구조는 설명을 위한 개념도입니다. 공식 문서에 따르면 Hive 스타일 파티션 디렉터리 구조는 **Delta Lake 프로토콜의 일부가 아니므로** 워크로드가 이 디렉터리 구조에 의존해서는 안 됩니다. 또한 column mapping을 켜면 파티션 디렉터리의 컬럼 이름 자리에 임의의 접두사가 들어갑니다.
{: .notice--info}

# 3. 만드는 법

```sql
CREATE TABLE main.l0.orders (
  order_dt  DATE,
  order_id  STRING,
  amount    DECIMAL(18,2),
  order_ym  STRING            -- 파티션용 파생 컬럼 (예: '2026-01')
)
PARTITIONED BY (order_ym);
```

```python
from pyspark.sql import functions as F

df = df.withColumn("order_ym", F.date_format("order_dt", "yyyy-MM"))   # 파티션 컬럼 만들기
(df.write.format("delta")
   .partitionBy("order_ym")
   .saveAsTable("main.l0.orders"))
```

## 제약

- 파티션 컬럼은 **최상위 컬럼**이어야 합니다. `struct_col.field` 같은 struct 필드는 파티션할 수 없습니다. (Delta가 `PARTITIONED BY`의 struct 필드를 컬럼 참조가 아닌 **식(expression)** 으로 취급하기 때문. struct 필드로 정리하려면 liquid clustering 사용)
- 지원 데이터 타입: Date, Timestamp, TimestampNTZ, Interval, String, Binary, Boolean, Integer/Long/Short/Byte, Float/Double/Decimal.
- **카디널리티(서로 다른 값의 개수)가 낮거나 알려진** 컬럼(날짜, 지역 등)에 적합합니다. timestamp처럼 값이 거의 매번 다른 컬럼에는 맞지 않습니다.

## 현재 상태 확인

```sql
DESCRIBE DETAIL main.l0.orders;
-- partitionColumns : 파티션 컬럼 목록 ([] 이면 파티션 없음)
-- numFiles, sizeInBytes : 파일 수와 전체 크기
```

# 4. 공식 문서의 현재 권장: "하지 말고, 해야 한다면 신중히"

공식 문서는 파티셔닝에 대해 **꽤 조심스러운 입장**입니다.

- **Databricks는 모든 관리형(managed) 테이블에 liquid clustering을 권장**합니다.
- "**100 TB 미만**인 대부분의 테이블은 파티셔닝이 필요 없다." Delta Lake는 파티션이 없는 테이블의 데이터를 **수집 시간(ingestion time) 기준으로 자동 클러스터링**해서, 직접 튜닝하지 않아도 파티셔닝과 비슷한 성능을 냅니다.
- 크기별 가이드:

| 테이블 데이터 크기 | 공식 권장 |
|---|---|
| **1 TB 미만** | 파티셔닝하지 말 것 |
| **1 TB ~ 100 TB** | 파티셔닝 대신 **liquid clustering**. 파티셔닝은 도움보다 성능을 해치는 경우가 더 많다 |
| **100 TB 이상** | 파티셔닝이 성능을 높일 수 있으나, **먼저 liquid clustering**을 쓰고 개선 여부를 확인 |

- 파티션을 만든다면 **각 파티션에 최소 1 GB 이상**의 데이터가 있는지 확인하라고 합니다. "크고 적은 수의 파티션"이 "작고 많은 파티션"보다 성능이 좋은 경향이 있습니다.

## 왜 잘못된 파티셔닝이 위험한가

> 공식 경고: 비효율적인 파티셔닝 전략은 쿼리 성능을 해칠 수 있고, **고치려면 데이터 전체를 다시 써야 할 수도 있습니다.** 큰 테이블에서는 매우 비싸고 느릴 수 있습니다.

- 파티션이 너무 잘게 나뉘면 **작은 파일이 폭증**(small files problem)해서 오히려 느려집니다. 예: 하루 10 MB씩 쌓이는 테이블을 일 단위로 파티셔닝하면 파티션 하나가 1 GB에 한참 못 미칩니다.
- 파티션 컬럼은 **나중에 바꾸기 어렵습니다.** 바꾸려면 전체 재작성이 필요합니다.

# 5. 다른 데이터 레이크와 다른 점 (Hive/Parquet 경험이 있다면)

공식 문서가 Hive·Parquet 경험자를 위해 짚어 주는 점입니다.

- **트랜잭션이 파티션 경계로 정의되지 않습니다.** Delta는 트랜잭션 로그로 ACID를 보장하므로, 원자성을 위해 배치를 파티션으로 나눌 필요가 없습니다.
- Databricks 컴퓨트 클러스터는 **데이터가 특정 디스크에 묶여 있는(data locality) 구조가 아닙니다.** 데이터는 클라우드 객체 스토리지에 있고, 파일 단위 통계로 읽을 데이터를 최소화합니다.
- 다른 오픈소스 환경에서 유효했던 파티셔닝 기법이 Databricks에서는 항상 맞지는 않습니다.

# 6. Z-order와의 관계

- Z-order는 `OPTIMIZE`가 필요하며, **파티션 경계를 넘어 파일을 합칠 수 없어서** 파티션 안에서만 클러스터링됩니다.
- **파티셔닝에 쓴 컬럼으로는 Z-order를 할 수 없습니다.**
- 파티셔닝은 카디널리티가 낮은 컬럼에만 잘 맞고, Z-order는 높은 카디널리티 컬럼에도 쓸 수 있습니다.
- 신규 테이블에는 Z-order보다 **liquid clustering**을 권장합니다.

# 7. 동시 쓰기와의 관계

- 서로 다른 파티션에 대한 작업은 **서로 다른 파일 집합**을 건드리므로, "충돌 가능" 쌍도 실제로는 충돌하지 않게 만들 수 있습니다. (조건 컬럼으로 파티셔닝)
- 반면 **파티션된 테이블은 Row-level concurrency를 쓸 수 없습니다.**
- 즉 "파티션 분리로 충돌을 피한다" vs "비파티션 + deletion vectors + liquid clustering으로 행 단위로 해결한다" 중 선택하는 문제입니다. (→ Row-level concurrency 글)

# 8. 파티션 덮어쓰기와의 관계

- **동적 파티션 덮어쓰기**(`partitionOverwriteMode=dynamic`): 새 데이터가 들어오는 파티션의 기존 데이터를 통째로 교체. 공식 문서는 **신규 워크로드에는 권장하지 않는** 레거시 방식으로 분류합니다. `replaceWhere`와는 **같은 쓰기에서 함께 쓸 수 없습니다.**
- **`replaceWhere`**: 조건에 맞는 행만 원자적으로 교체. **파티션 컬럼이 아니어도** 쓸 수 있습니다. (→ replaceWhere 글)
- 공식 경고: 파티션 기반 덮어쓰기에서는 **"엉뚱한 파티션에 속한 행 하나"가 파티션 전체를 덮어쓸 수 있으므로**, 쓰는 데이터가 의도한 파티션만 건드리는지 항상 검증하세요.

# 9. 의사결정 가이드

```
테이블 크기가 어느 정도인가?
 ├─ 1 TB 미만 ─────────────→ 파티셔닝 하지 않는다 → liquid clustering(권장) 또는 파티션 없음
 ├─ 1 ~ 100 TB ────────────→ liquid clustering
 └─ 100 TB 이상
      ├─ 파티션 1개가 최소 1 GB 이상 되는 컬럼이 있는가?
      │    ├─ 예 → 파티셔닝 검토 (단, 먼저 liquid clustering 으로 비교)
      │    └─ 아니오 → liquid clustering
```

## 이미 파티션된 테이블이라면

`ALTER TABLE ... REPLACE PARTITIONED BY WITH CLUSTER BY` (DBR 18.1 이상)로 liquid clustering으로 변환할 수 있습니다. (→ Liquid Clustering 글)

# 10. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "큰 테이블이면 일단 파티셔닝" | 공식 권장은 **liquid clustering 우선**. 파티셔닝은 100 TB 이상에서도 LC를 먼저 비교 |
| "월별 청크로 적재하니 월별 파티셔닝해야 한다" | **읽기 청크(JDBC 파티션)와 저장 파티셔닝은 별개**. 청크 기준이 월이어도 테이블 레이아웃은 따로 결정 |
| "파티션을 나눠야 원자적 쓰기가 된다" | Delta는 트랜잭션 로그로 원자성을 보장. 파티션과 무관 |
| "파티션 많을수록 빠르다" | 파티션이 작아지면 작은 파일이 늘어 **오히려 느려짐** (파티션당 1 GB 이상 권장) |
| "나중에 바꾸면 되지" | 파티션 변경은 **전체 재작성**이 필요할 수 있음 |

# 11. 한 줄 정리

- 파티셔닝 = **컬럼 값별로 파일을 나눠 저장**해 필요한 부분만 읽게 하는 방식.
- 공식 문서는 **1 TB 미만이면 하지 말고**, 모든 테이블에 **liquid clustering을 우선** 권장한다.
- 하게 되면 **파티션당 최소 1 GB**, **낮은 카디널리티** 컬럼, **최상위 컬럼**만.
- 파티션 테이블은 row-level concurrency를 못 쓰고, 파티션 변경은 전체 재작성이 필요할 수 있다.

# 관련 용어
Liquid Clustering, Row-level concurrency, JDBC 읽기 파티션, replaceWhere

## 🔗 참고
- [When to partition tables on Databricks](https://docs.databricks.com/aws/en/tables/partitions)
- [Use liquid clustering for tables](https://docs.databricks.com/aws/en/delta/clustering)
- [Selectively overwrite data with Delta Lake](https://docs.databricks.com/aws/en/delta/selective-overwrite)
- [Row-level concurrency](https://docs.databricks.com/aws/en/optimizations/isolation/row-level-concurrency)
