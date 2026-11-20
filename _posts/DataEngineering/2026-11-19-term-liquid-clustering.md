---
title:  "[DE/용어] Liquid Clustering"
excerpt: "파티셔닝과 ZORDER를 대체하는 레이아웃 기능을 개념부터 전환·키 선택까지"

categories:
  - DataEngineering
tags:
  - [Glossary, Delta Lake, Liquid Clustering, Data Layout]

toc: true
toc_sticky: true
published: true

date: 2026-11-19
last_modified_at: 2026-11-19
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-19)    
[Use liquid clustering for tables](https://docs.databricks.com/aws/en/delta/clustering)    
[When to partition tables on Databricks](https://docs.databricks.com/aws/en/tables/partitions)
{: .notice--warning}

# 0. 이 글을 읽기 전에

"데이터 레이아웃(data layout)"이란 **같은 데이터를 어떤 파일에 어떻게 나눠 담을지**를 뜻합니다. 레이아웃이 좋으면 쿼리가 필요한 파일만 읽고 나머지를 건너뛸 수 있어 빨라집니다. 이 글은 그 방법 중 Databricks가 현재 권장하는 **Liquid Clustering**을 다룹니다. **Delta 테이블 파티셔닝** 글을 먼저 보시면 비교가 쉽습니다.

# 1. 한 줄 정의

**클러스터링 키**로 지정한 컬럼을 기준으로 데이터를 자동으로 정리해 쿼리 성능을 높이는 데이터 레이아웃 최적화 기능입니다. **테이블 파티셔닝과 ZORDER를 대체**합니다. 선언은 `CLUSTER BY`로 합니다.

# 2. 먼저 알아야 할 개념: 데이터 스키핑(data skipping)

Delta는 각 데이터 파일에 대해 **컬럼별 통계(최솟값/최댓값 등)** 를 로그에 기록해 둡니다. 쿼리 조건이 어떤 파일의 범위와 겹치지 않으면 **그 파일은 읽지 않고 건너뜁니다.** 이것이 데이터 스키핑입니다.

```
[정리되지 않은 레이아웃]  order_dt 가 파일마다 뒤섞임
 file_1: order_dt 최소 01-02 ~ 최대 12-30
 file_2: order_dt 최소 01-05 ~ 최대 12-28
 file_3: order_dt 최소 01-01 ~ 최대 12-31
 → WHERE order_dt = '2026-06-10' : 세 파일 모두 범위에 걸려 전부 읽어야 함

[클러스터링된 레이아웃]  비슷한 order_dt 끼리 같은 파일에 모음
 file_A: 01-01 ~ 04-30
 file_B: 05-01 ~ 08-31
 file_C: 09-01 ~ 12-31
 → WHERE order_dt = '2026-06-10' : file_B 만 읽고 A, C 는 건너뜀
```

Liquid clustering은 **"비슷한 값끼리 같은 파일에 모이도록" 데이터를 정리**해서 데이터 스키핑 효과를 높입니다. (위 숫자는 개념 설명용 예시)

# 3. 왜 파티셔닝 대신 쓰는가

| | 파티셔닝 | Liquid clustering |
|---|---|---|
| 나누는 방식 | 값별 **디렉터리**로 고정 | 키 기준으로 **유연하게** 파일 정리 |
| 카디널리티 | 낮아야 함 | **낮은/높은 카디널리티 모두 가능** |
| 작은 파일 문제 | 자주 발생 | 고정 파티션 경계와 작은 파일 문제를 **피함** |
| 키 변경 | **어렵다** (전체 재작성 필요할 수 있음) | **쉽다** (기존 데이터를 다시 쓰지 않고 변경) |
| 동시 쓰기 | row-level concurrency 불가 | **row-level concurrency 허용** → 충돌 감소 |
| 공식 권장 | 1 TB 미만 비권장 | **모든 신규 테이블에 권장** |

공식 문서가 liquid clustering의 이점이 큰 경우로 꼽는 상황입니다.

- **높은 카디널리티** 컬럼으로 자주 필터하는 쿼리
- 데이터 **쏠림(skew)** 이 심한 테이블
- 빠르게 커져서 유지보수·튜닝 부담이 큰 테이블
- **동시 쓰기**가 필요한 테이블
- 접근 패턴이 **다양하거나 바뀌는** 테이블
- 일반적인 파티션 키가 **너무 많거나 너무 적은** 파티션을 반환하는 테이블

# 4. 만드는 법

## 4-1. 새 테이블을 만들 때

```sql
CREATE TABLE main.l0.orders (
  order_dt    DATE,
  order_id    STRING,
  customer_id STRING,
  amount      DECIMAL(18,2)
)
CLUSTER BY (order_dt);
```

- **CTAS**(`CREATE TABLE ... AS SELECT`)에서는 `CLUSTER BY`가 **테이블 이름 뒤**에 와야 하고, SELECT 절 안에 쓰면 안 됩니다.

```sql
CREATE TABLE main.l0.orders_copy CLUSTER BY (order_dt) AS SELECT * FROM main.l0.orders;
```

- 구조만 복사(클러스터링 설정 포함): `CREATE TABLE main.l0.orders2 LIKE main.l0.orders;`

## 4-2. PySpark / Scala

```python
# DataFrame 에서 바로 저장
df.write.clusterBy("order_dt").saveAsTable("main.l0.orders")

# DataFrameWriterV2 (DBR 14.2 이상)
df.writeTo("main.l0.orders").using("delta").clusterBy("order_dt").create()

# DeltaTable API: 빈 테이블 만들기
from delta.tables import DeltaTable
(DeltaTable.create(spark)
   .tableName("main.l0.orders")
   .addColumn("order_dt", "DATE")
   .addColumn("order_id", "STRING")
   .clusterBy("order_dt")
   .execute())

# Structured Streaming (DBR 16.4 LTS 이상)
(spark.readStream.table("source_table")
   .writeStream.clusterBy("order_dt")
   .option("checkpointLocation", checkpoint_path)
   .toTable("target_table"))
```

## 4-3. 이미 있는 (파티션 없는) 테이블에 켜기

```sql
ALTER TABLE main.l0.orders CLUSTER BY (order_dt);
```

## 4-4. 키 제거

```sql
ALTER TABLE main.l0.orders CLUSTER BY NONE;
```

# 5. 켜면 데이터는 언제 정리되는가 (중요)

가장 흔한 오해입니다. **`CLUSTER BY`를 설정한다고 기존 데이터가 즉시 정리되지는 않습니다.**

- 기본 동작은 **이미 쓰여 있던 데이터에는 클러스터링을 적용하지 않습니다.**
- 켠 뒤에 **`OPTIMIZE`를 실행**해야 점진적으로 클러스터링됩니다.
- 이전 데이터까지 **강제로 재클러스터링**하려면 `OPTIMIZE <table> FULL` 또는 `OPTIMIZE <table> FULL WHERE <조건>`을 씁니다.
- 클러스터링 키를 **바꾼 경우**에도 변경의 효과를 보려면 `OPTIMIZE`를 실행해야 합니다.
- 공식 문서는 가능하면 **플랫폼이 모든 레이아웃·최적화 작업을 관리**하게 두라고 권장하며(예: Unity Catalog 관리형 테이블의 predictive optimization), 자동 키 선택은 `CLUSTER BY AUTO`로 사용할 수 있다고 안내합니다. (자동 liquid clustering은 Unity Catalog 관리형 테이블에서만 가능)

```sql
OPTIMIZE main.l0.orders;                       -- 점진적 클러스터링
OPTIMIZE main.l0.orders FULL;                  -- 전체 강제 재클러스터링
OPTIMIZE main.l0.orders FULL WHERE order_dt >= '2026-01-01';   -- 일부만 강제
```

# 6. 클러스터링 키 고르기

## 6-1. 원칙

- **쿼리 필터(WHERE)에 가장 자주 쓰는 컬럼**을 고릅니다.
- 순서는 상관없이 지정할 수 있습니다(필요하면 계층적 클러스터링으로 우선순위 부여).
- 서로 **상관관계가 높은** 두 컬럼은 **하나만** 포함하면 됩니다.
- 키는 **최대 4개**까지 지정할 수 있습니다.
- **10 TB 미만**의 작은 테이블에서는 키를 많이 두면 **단일 컬럼으로 필터할 때 성능이 떨어질 수 있습니다.** (예: 키 4개로 필터링하는 것이 키 2개보다 느릴 수 있음. 테이블이 커질수록 이 차이는 무시할 만해짐)
- 키는 **통계가 수집되는 컬럼**이어야 합니다. Delta 테이블은 기본적으로 **처음 32개 컬럼**에 대해 통계를 수집합니다.

## 6-2. 지원 데이터 타입

Date, Timestamp, TimestampNTZ(DBR 14.3 LTS 이상), String, Integer/Long/Short/Byte, Float/Double/Decimal.

- struct 필드는 점 표기법으로 가능합니다: `CLUSTER BY (struct_col.field)`, 중첩도 가능(`struct_col.nested.field`). 단 필드의 타입이 위 목록 중 하나여야 합니다.
- **불가**: StructType/MapType/ArrayType 자체, map/array의 요소(`map_col['key']`, `array_col[0]`).

## 6-3. 우리 사례(월별 적재 테이블)의 예

- 조회가 주로 `WHERE order_dt BETWEEN ...` 이라면 키: `order_dt`
- 고객 단위 조회도 많다면 `CLUSTER BY (order_dt, customer_id)`
- 키가 많다고 좋은 것이 아니므로, 실제 쿼리 패턴을 확인한 뒤 정하세요.

# 7. 파티션·ZORDER에서 전환

## 7-1. 기존 최적화 방식별 권장 키

| 현재 쓰는 방식 | 클러스터링 키 권장 |
|---|---|
| Hive 스타일 파티셔닝 | **파티션 컬럼**을 클러스터링 키로 |
| Z-order | **ZORDER BY 컬럼**을 클러스터링 키로 |
| 파티셔닝 + Z-order | **둘 다** 클러스터링 키로 |
| 카디널리티를 줄이려고 만든 **생성(generated) 컬럼** (예: timestamp에서 date) | **원래 컬럼**을 키로 쓰고, 생성 컬럼은 만들지 않음 |

## 7-2. 자동 변환 명령 (DBR 18.1 이상)

```sql
-- 파티션 컬럼을 그대로 클러스터링 키로 사용
ALTER TABLE t3 REPLACE PARTITIONED BY WITH CLUSTER BY;

-- 다른 컬럼으로 (예: (year, month, day) 로 파티션된 테이블)
ALTER TABLE t1 REPLACE PARTITIONED BY WITH CLUSTER BY (day, id);
OPTIMIZE t1;        -- 키를 바꿨으니 OPTIMIZE 필요

-- AUTO: 현재 파티션 컬럼으로 시작해서 predictive optimization 이 시간이 지나며 조정
ALTER TABLE t2 REPLACE PARTITIONED BY WITH CLUSTER BY AUTO;   -- Unity Catalog 관리형 테이블만
```

- 새 키를 **원래 파티션 컬럼과 비슷하게** 두라고 권장합니다. 많이 다른 컬럼을 쓰면 첫 `OPTIMIZE`에서 **대규모 재클러스터링**이 일어납니다.
- 변환 후에는 DBR 13.3 LTS 이상에서 읽기·쓰기가 가능합니다. 변환 **중** 읽고 쓰는 워크로드가 있다면 **DBR 15.4 LTS 이상**을 권장합니다. 배치 쓰기는 15.4 이상에서 다운타임이 없고, 15.3 이하에서는 변환 전 워크로드를 멈추고 변환 후 재시작하라고 안내합니다.
- **timestamp 컬럼**으로 파티션된 테이블을 변환할 때는 추가 설정이 필요합니다.

```sql
SET spark.databricks.delta.liquidConversion.statsGeneration.enabled = false;
ALTER TABLE t1 REPLACE PARTITIONED BY WITH CLUSTER BY (timestamp_col, id);
ANALYZE TABLE t1 COMPUTE DELTA STATISTICS;
```

## 7-3. 변환 확인과 되돌리기

- 확인: `DESCRIBE EXTENDED`로 새 클러스터링 컬럼 확인, `DESCRIBE HISTORY`로 변환 관련 작업 이력(REORG, UPGRADE PROTOCOL, REPLACE PARTITIONED BY WITH CLUSTER BY) 확인
- 되돌리기: (a) CTAS로 테이블 재생성(변환 이후의 데이터 변경 유지, 전체 재작성) 또는 (b) 변환 이전 버전으로 `RESTORE`(이후 변경 모두 폐기). 롤백 전 쓰기를 멈추고, `TRUNCATE` 직후 `RESTORE`를 실행해 VACUUM에 의한 파일 삭제를 막으라는 경고가 있습니다.

## 7-4. 변환 제한

- Lakeflow 파이프라인의 streaming table·materialized view는 이 명령으로 변환할 수 없고, 파이프라인 정의를 `PARTITIONED BY` 대신 `CLUSTER BY`로 바꿔야 합니다.
- 파티션 필터링을 쓰는 Delta Sharing 테이블은 지원되지 않습니다.

# 8. 제약과 주의 (꼭 읽어 보세요)

- **파티셔닝, ZORDER와 함께 쓸 수 없습니다.**
- Delta 테이블에서의 GA: **Databricks Runtime 15.4 LTS 이상** (최신 런타임 권장). 클러스터 런타임이 이보다 낮은지 확인하세요.
- **프로토콜 경고**: liquid clustering이 켜진 Delta 테이블은 **Delta writer 버전 7, reader 버전 3**을 사용합니다. 이를 지원하지 않는 Delta 클라이언트는 **읽을 수 없고**, 프로토콜 버전을 **낮출(downgrade) 수 없습니다.**
- 제거하려면 `CLUSTER BY NONE`으로 키를 없앨 수 있지만, 이미 올라간 프로토콜은 되돌릴 수 없다는 점을 기억하세요.

# 9. 동시성과의 관계

- 공식 문서는 liquid clustering 테이블이 **row-level concurrency를 허용해 쓰기 충돌이 줄어든다**고 설명합니다.
- 월별 청크를 동시에 `replaceWhere`로 쓰는 경우에도, 파티션 테이블보다 충돌 부담이 적을 수 있습니다. 다만 `replaceWhere` 동시 실행의 충돌 여부는 공식 충돌 표에 별도 행이 없으니 **실제로 테스트**하세요. (→ Row-level concurrency 글)

# 10. 파티셔닝 vs Liquid Clustering 선택 가이드

```
신규 테이블인가?
 ├─ 예 → Liquid clustering 권장 (Delta 테이블 GA: DBR 15.4 LTS+)
 │        └─ 클러스터가 15.4 LTS 미만이고 업그레이드 불가 → 파티셔닝 또는 파티션 없음 검토
 └─ 아니오 (이미 파티션됨)
          ├─ DBR 18.1+ → REPLACE PARTITIONED BY WITH CLUSTER BY 로 변환 검토
          └─ 그 외 → 현 상태 유지 후 업그레이드 계획
외부 도구가 이 테이블을 읽는가? → 프로토콜(writer 7 / reader 3) 호환성 확인 필수
```

# 11. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "`CLUSTER BY`만 하면 기존 데이터도 바로 정리된다" | 기존 데이터에는 적용되지 않음. `OPTIMIZE`(필요 시 `FULL`) 실행 필요 |
| "키는 많을수록 좋다" | 최대 4개. 작은 테이블에서는 단일 컬럼 필터 성능이 **떨어질 수 있음** |
| "아무 컬럼이나 키가 된다" | **통계가 수집되는 컬럼**(기본 처음 32개)이고 지원 타입이어야 함 |
| "파티션과 같이 쓰면 더 좋다" | **같이 쓸 수 없음** |
| "켰다가 언제든 원복 가능" | 프로토콜이 올라가면 **다운그레이드 불가** |

# 12. 한 줄 정리

- 같은 값(정확히는 비슷한 값)끼리 같은 파일에 모아 **데이터 스키핑**을 극대화하는 레이아웃 기능.
- 파티셔닝·ZORDER를 **대체**하며, 공식 문서는 **모든 신규 테이블에 권장**한다.
- `CLUSTER BY`로 선언하고, 기존 데이터에는 **`OPTIMIZE`** 를 실행해야 반영된다.
- 키는 **필터에 자주 쓰는 컬럼, 최대 4개, 통계 수집 컬럼**.
- Delta 테이블 GA는 **DBR 15.4 LTS+**, **프로토콜이 올라가므로 외부 클라이언트 호환성과 다운그레이드 불가**에 주의.
- **row-level concurrency를 허용**해 동시 쓰기 충돌이 줄어든다.

# 관련 용어
테이블 파티셔닝, Row-level concurrency, Deletion vectors, replaceWhere

## 🔗 참고
- [Use liquid clustering for tables](https://docs.databricks.com/aws/en/delta/clustering)
- [When to partition tables on Databricks](https://docs.databricks.com/aws/en/tables/partitions)
- [Row-level concurrency](https://docs.databricks.com/aws/en/optimizations/isolation/row-level-concurrency)
