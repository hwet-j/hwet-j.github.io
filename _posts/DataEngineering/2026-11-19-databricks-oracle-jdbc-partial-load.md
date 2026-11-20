---
title:  "[DE/Databricks] Oracle → Delta 병렬 적재, 일부 청크가 실패해도 성공분은 살리기"
excerpt: "Spark 쓰기 한 번 = Delta 트랜잭션 한 번. 그래서 하나만 실패해도 전체가 롤백되는 구조와, 청크 단위로 쪼개서 적재하는 방법"

categories:
  - DataEngineering
tags:
  - [Databricks, Delta Lake, Oracle, JDBC, replaceWhere, Lakeflow Jobs]

toc: true
toc_sticky: true
published: true

date: 2026-11-19
last_modified_at: 2026-11-19
---

{% raw %}

업무에서 겪은 상황을 일반화해서 정리했습니다. 테이블명·컬럼명은 모두 **예시**이며 실제 이름이 아닙니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-19)    
[Isolation levels](https://docs.databricks.com/aws/en/optimizations/isolation/isolation-levels) ·
[Row-level concurrency](https://docs.databricks.com/aws/en/optimizations/isolation/row-level-concurrency) ·
[Selective overwrite](https://docs.databricks.com/aws/en/delta/selective-overwrite) ·
[Partitions](https://docs.databricks.com/aws/en/tables/partitions) ·
[Liquid clustering](https://docs.databricks.com/aws/en/delta/clustering) ·
[For each task](https://docs.databricks.com/aws/en/jobs/tasks/for-each) ·
[Repair job failures](https://docs.databricks.com/aws/en/jobs/repair-job-failures) ·
[Spark JDBC](https://spark.apache.org/docs/latest/sql-data-sources-jdbc.html) ·
[Spark configuration](https://spark.apache.org/docs/latest/configuration.html)
{: .notice--warning}

# 1. 상황

Oracle 원천 테이블을 Databricks의 Delta 테이블(L0)로 적재하는 배치입니다.

| 항목 | 내용 (예시) |
|---|---|
| 원천 | Oracle `SALES.ORDERS` |
| 대상 | Delta `main.l0.orders` |
| 기준 컬럼 | `order_dt` (주문일) |
| 읽기 방식 | 기준 컬럼 기준 **월 단위로 쪼개서 병렬 조회** (`spark.read.jdbc(predicates=...)`) |
| 적재 방식 | 이번에 처리할 기간의 기존 데이터를 **지우고(DELETE) 다시 적재(append)** |

재적재 방식을 택한 이유는 **운영 실수에 대비**하기 위해서입니다. 같은 기간을 몇 번 다시 돌려도 중복이 생기지 않아야 하기 때문입니다.

# 2. 증상

병렬로 읽는 월 중 **한 달의 조회가 실패하면 전체 적재가 실패**합니다. 나머지 11개월은 정상 조회됐는데도 Delta에는 아무것도 들어가지 않습니다.

> 원하는 동작: 월별로 읽는 대로 저장되고, 실패한 달만 다시 돌린다.

# 3. 원인: 쓰기 한 번 = 트랜잭션 한 번

```
predicates (12개)          Spark                     Delta
 2026-01 ─┐
 2026-02 ─┤                                    ┌───────────────┐
   ...    ├─> 하나의 DataFrame ─> write 1회 ─> │ 커밋 1번(마지막) │
 2026-12 ─┘   (파티션 12개)        (job 1개)    └───────────────┘
                                   한 task라도 실패하면 → 커밋 없음(전체 롤백)
```

- `jdbc(predicates=...)`는 predicate 하나가 DataFrame **파티션 하나**가 되어 병렬로 읽힙니다. 하지만 결과는 **DataFrame 하나**입니다. ([PySpark API](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrameReader.jdbc.html): "each one defines one partition of the DataFrame")
- `write`는 **하나의 Spark job**이고, Delta는 이 job이 끝날 때 **한 번 커밋**합니다.
- 특정 task가 `spark.task.maxFailures`(기본 4, 재시도 3회)만큼 **연속으로** 실패하면 Spark는 job을 포기합니다. 다른 task의 실패는 합산되지 않고, 한 번 성공하면 카운트가 초기화됩니다. ([Spark configuration](https://spark.apache.org/docs/latest/configuration.html))
- job이 실패하면 커밋이 없으니 **아무것도 반영되지 않습니다.** 이건 버그가 아니라 Delta의 원자성(ACID)입니다.

그래서 "에러 없이 병렬로 읽을 때마다 저장"하려면, 쓰기를 **여러 번의 독립된 커밋으로 쪼개는** 수밖에 없습니다. `spark.task.maxFailures`를 올리면 일시적인 Oracle 오류는 줄일 수 있지만, 부분 적재가 되는 것은 아닙니다.

# 4. 기존 코드 분석

아래는 문제가 되는 부분만 일반화한 코드입니다.

```python
# (1) 월별 predicate로 읽기 → DataFrame 하나
extract_df = spark.read.jdbc(
    url=jdbc_url, table=query, predicates=predicates, properties=conn_props
)

# (2) 컬럼명 소문자 변환
for column_name in extract_df.columns:
    transform_df = extract_df.withColumnRenamed(column_name, column_name.lower())

# (3) 기존 범위 삭제 → append
spark.sql(f"DELETE FROM {target_table} WHERE {delete_condition}")     # 커밋 ①
write_df(transform_df, mode="append", table=target_table)             # 커밋 ② (실패 가능)
```

## 문제 ① 한 번의 쓰기라서 전부 롤백

위 3장의 구조 때문입니다. 한 달 실패 = 전체 실패.

## 문제 ② DELETE와 append가 서로 다른 커밋 (가장 위험)

```
커밋 ①  DELETE 2026-01 ~ 2026-12   ← 이미 확정됨
커밋 ②  append                     ← 여기서 실패하면?
결과    대상 테이블의 해당 기간이 비어 있음
```

DELETE는 별도 트랜잭션으로 **이미 커밋**됩니다. 이후 append가 실패하면 기존 데이터까지 사라진 채로 남습니다. 재실행하면 복구되지만, 그 전까지 다운스트림은 빈 데이터를 읽습니다.

"지우고 다시 적재"라는 설계 의도는 맞습니다. 다만 **한 번의 원자적 커밋**으로 해야 합니다(5장의 `replaceWhere`).

## 문제 ③ 컬럼 소문자 변환 루프 버그

```python
for column_name in extract_df.columns:
    transform_df = extract_df.withColumnRenamed(column_name, column_name.lower())
```

매 반복마다 `extract_df`에서 **다시 시작**해서, 결과적으로 **마지막 컬럼 하나만** 소문자가 됩니다(Oracle 컬럼은 보통 대문자). 한 번에 바꾸면 됩니다.

```python
transform_df = extract_df.toDF(*[c.lower() for c in extract_df.columns])
```

# 5. 개선: 월별 청크 + `replaceWhere`

## 5-1. 핵심 아이디어

| | 기존 | 개선 |
|---|---|---|
| 읽기 | 12개 predicate → DataFrame 1개 | 월 1개 → DataFrame 1개 (12번) |
| 쓰기 | 12개월 한 번에 | **월마다 따로** |
| 삭제 + 적재 | DELETE 커밋 + append 커밋 | `replaceWhere` **원자적 커밋 1번** |
| 한 달 실패 시 | 전체 실패 | 그 달만 실패, 나머지는 반영 |
| 재실행 | 전체 | 실패한 달만 (멱등) |

`replaceWhere`는 조건에 맞는 행만 **원자적으로 덮어쓰며**, 테이블이 그 컬럼으로 파티션되어 있을 필요가 없습니다. 쓰는 데이터가 조건에 맞지 않으면 기본적으로 오류로 중단됩니다. (Python/Scala DBR 9.1 LTS 이상, SQL 12.2 LTS 이상) ([Selective overwrite](https://docs.databricks.com/aws/en/delta/selective-overwrite))

## 5-2. 청크(월) 만들기

Oracle용 조건과 Delta용 조건을 **같은 범위**로 함께 만듭니다. 상한은 `<`(미만)으로 두면 `DATE`/`TIMESTAMP` 어느 쪽이든 안전합니다.

```python
from dataclasses import dataclass
from datetime import date

@dataclass(frozen=True)
class Chunk:
    label: str          # "2026-01"
    oracle_cond: str    # Oracle WHERE 절용
    delta_cond: str     # Delta replaceWhere용

def month_chunks(from_dt: date, to_dt: date):
    """from_dt ~ to_dt 가 걸치는 월을 월 단위 청크로 분할"""
    y, m = from_dt.year, from_dt.month
    while date(y, m, 1) <= to_dt:
        start = date(y, m, 1)
        ny, nm = (y + 1, 1) if m == 12 else (y, m + 1)
        end = date(ny, nm, 1)
        yield Chunk(
            label=start.strftime("%Y-%m"),
            oracle_cond=(f"order_dt >= TO_DATE('{start}', 'YYYY-MM-DD') "
                         f"AND order_dt < TO_DATE('{end}', 'YYYY-MM-DD')"),
            delta_cond=f"order_dt >= '{start}' AND order_dt < '{end}'",
        )
        y, m = ny, nm
```

> 기존 `build_read_plan`이 이미 월별 predicate를 만들어 준다면, 그 함수가 **같은 범위의 Delta 조건도 함께 반환**하도록만 바꿔도 됩니다.
{: .notice--info}

## 5-3. 청크 하나를 읽고 쓰는 함수

```python
from pyspark.sql import functions as F
from pyspark.sql.types import TimestampType

SRC = "SALES.ORDERS"
TARGET = "main.l0.orders"
COLS = "order_dt, order_id, customer_id, status_cd, amount, load_dttm"

def load_chunk(chunk: Chunk, allow_empty_delete: bool = False) -> dict:
    query = f"(SELECT {COLS} FROM {SRC} WHERE {chunk.oracle_cond}) src"

    df = spark.read.jdbc(url=jdbc_url, table=query, properties=conn_props)

    # 컬럼명 소문자 (한 번에)
    df = df.toDF(*[c.lower() for c in df.columns])

    # TIMESTAMP -> TIMESTAMP_NTZ
    for f in df.schema.fields:
        if isinstance(f.dataType, TimestampType):
            df = df.withColumn(f.name, F.col(f.name).cast("timestamp_ntz"))

    # 빈 청크 처리 (아래 6-3 참고)
    if df.limit(1).isEmpty() and not allow_empty_delete:
        print(f"[WARN] {chunk.label}: 원천 0건 -> 기존 데이터 유지(skip)")
        return {"label": chunk.label, "status": "skipped_empty"}

    # 삭제 + 적재를 원자적 커밋 1번으로
    (df.write
        .format("delta")
        .mode("overwrite")
        .option("replaceWhere", chunk.delta_cond)
        .saveAsTable(TARGET))

    # 건수는 원천 재조회 없이 Delta 에서 (해당 청크 범위만)
    cnt = spark.table(TARGET).where(chunk.delta_cond).count()
    return {"label": chunk.label, "status": "ok", "rows": cnt}
```

## 5-4. 병렬 실행 + 실패 수집 + 재시도

```python
import time
from concurrent.futures import ThreadPoolExecutor, as_completed

def run_chunks(chunks, max_workers=3, max_attempts=2):
    ok, failed = [], {}

    def worker(ch):
        last = None
        for attempt in range(1, max_attempts + 1):
            try:
                return load_chunk(ch)
            except Exception as e:
                last = e
                print(f"[WARN] {ch.label} attempt {attempt}/{max_attempts} failed: {e}")
                time.sleep(10 * attempt)          # 짧은 backoff
        raise last

    with ThreadPoolExecutor(max_workers=max_workers) as ex:
        futs = {ex.submit(worker, ch): ch for ch in chunks}
        for fut in as_completed(futs):
            ch = futs[fut]
            try:
                ok.append(fut.result())
            except Exception as e:
                failed[ch.label] = repr(e)[:300]
    return ok, failed

chunks = list(month_chunks(from_date, to_date))
ok, failed = run_chunks(chunks, max_workers=3)

total_rows = sum(r.get("rows", 0) for r in ok)
dbutils.jobs.taskValues.set("data_count", total_rows)
dbutils.jobs.taskValues.set("failed_chunks", sorted(failed))

print(f"[INFO] success={len(ok)}, failed={len(failed)}, rows={total_rows}")
if failed:
    # 성공한 청크는 이미 반영됨. Job 상태는 실패로 남겨서 알림/재실행 유도
    raise RuntimeError(f"failed chunks: {failed}")
```

- `max_workers`는 **동시에 열리는 Oracle 커넥션 수**와 같다고 보면 됩니다(청크당 연결 1개). Oracle 부하와 클러스터 코어를 보고 정하세요.
- 재실행 때는 `failed_chunks`에 나온 월만 `chunks`로 넘기면 됩니다. 쓰기가 멱등(`replaceWhere`)이라 중복이 생기지 않습니다.
- Spark는 여러 스레드에서의 job 제출을 지원하므로 노트북 안에서 `ThreadPoolExecutor`로 병렬 실행할 수 있습니다.

## 5-5. 기준 컬럼이 없는 테이블

기존 코드의 "기준 컬럼이 없거나 테이블이 없으면 전체 `overwrite`" 분기는 그대로 둡니다. 청크 분할이 불가능하므로 이 경우는 부분 적재가 의미가 없습니다.

# 6. 상황별 선택

## 6-1. 병렬 실행을 어디서 할까

| 상황 | 추천 | 이유 |
|---|---|---|
| 청크가 수십 개 이하, 노트북 하나로 충분 | **스레드**(5-4) | 구현이 가장 단순 |
| 청크별 개별 재시도·알림·이력이 필요 | **For each task** | UI에서 반복별 상태 확인, 반복별 retry |
| 실패분만 사람이 확인 후 다시 돌리고 싶음 | **For each + Repair run** | 실패한 task와 하위 task만 재실행 |

For each를 쓸 때의 공식 문서 요점입니다. ([For each task](https://docs.databricks.com/aws/en/jobs/tasks/for-each), [Repair](https://docs.databricks.com/aws/en/jobs/repair-job-failures))

- 입력은 객체 배열이고, 각 객체가 반복 1회에 전달됩니다(`{{input.label}}` 등으로 참조).
- **Concurrency**로 동시 실행 수를 정합니다(기본값 1).
- 재시도는 task의 `max_retries`, `min_retry_interval_millis` 등 일반 task 설정으로 둡니다.
- Repair는 **실패한 task를 처음부터** 다시 실행합니다. Jobs는 task의 멱등성을 보장하지 않으므로, 일부만 쓰고 실패했을 때 중복이 생기지 않게 **overwrite/merge 같은 멱등한 쓰기**를 써야 합니다. → `replaceWhere`가 바로 그 용도입니다.
- Repair는 task가 2개 이상인 Job에서만 지원됩니다.

```python
# For each 의 nested task(노트북)에서
import json
chunk_input = json.loads(dbutils.widgets.get("chunk"))   # {"label":"2026-01","from":"2026-01-01","to":"2026-02-01"}
# → 5-3 의 load_chunk 를 이 청크 하나에 대해서만 호출
```

## 6-2. 테이블 레이아웃: 파티셔닝 vs Liquid Clustering

먼저 용어를 구분해야 합니다.

| | JDBC 읽기 파티션 | Delta 테이블 파티셔닝 |
|---|---|---|
| 무엇인가 | Oracle에서 **읽는** 단위 | Delta 테이블 파일의 **저장** 방식 |
| 설정 위치 | `predicates` (월별 조건) | 테이블 생성 시 `PARTITIONED BY` |

월별로 조회하는 것은 **읽기 파티션**입니다. 여기서 정할 것은 **대상 Delta 테이블의 저장 레이아웃**입니다. 현재 상태는 아래로 확인합니다.

```sql
DESCRIBE DETAIL main.l0.orders;   -- partitionColumns, clusteringColumns, sizeInBytes, numFiles
```

**A. 전통적인 파티셔닝**

```sql
CREATE TABLE main.l0.orders (
  order_dt DATE, order_id STRING, customer_id STRING,
  status_cd STRING, amount DECIMAL(18,2), load_dttm TIMESTAMP_NTZ,
  order_ym STRING            -- 파티션용 파생 컬럼 (예: 'yyyy-MM')
)
PARTITIONED BY (order_ym);
```

**B. Liquid clustering (공식 문서 권장)**

```sql
CREATE TABLE main.l0.orders (
  order_dt DATE, order_id STRING, customer_id STRING,
  status_cd STRING, amount DECIMAL(18,2), load_dttm TIMESTAMP_NTZ
)
CLUSTER BY (order_dt);
```

```python
# DataFrame 으로 처음 만들 때
df.write.clusterBy("order_dt").saveAsTable("main.l0.orders")
```

| | 파티셔닝 | Liquid clustering |
|---|---|---|
| 문서 권장 | **1 TB 미만이면 하지 말 것.** 파티션당 1 GB 이상 확인 | **모든 신규 테이블에 권장** |
| 키 변경 | 어렵다(데이터 재작성) | 데이터 재작성 없이 변경 |
| 병용 | LC·ZORDER와 같이 못 씀 | 파티셔닝·ZORDER와 같이 못 씀 |
| 최소 버전 | - | Delta 테이블 GA: DBR 15.4 LTS 이상 |

- 기존 파티션 테이블은 DBR 18.1 이상에서 `ALTER TABLE ... REPLACE PARTITIONED BY WITH CLUSTER BY`로 변환할 수 있습니다.
- 월별 파티션 하나가 1 GB보다 작다면, 파티셔닝은 작은 파일만 늘려 오히려 느려질 수 있습니다.

**선택 기준**

```
테이블 < 1 TB                      → Liquid clustering (CLUSTER BY order_dt)
1 TB 이상 & 월 파티션 충분히 큼      → 파티셔닝도 선택지 (LC 먼저 검토 권장)
DBR 15.4 LTS 미만 클러스터          → 파티셔닝 또는 런타임 업그레이드
```

테이블은 **미리 DDL로 만들어 두는 것**을 권장합니다(레이아웃을 의도대로 고정, 첫 실행 분기 불필요).

## 6-3. 빈 청크(원천 0건)는 어떻게 할까

공식 문서에 따르면 소스가 비어 있을 때 `REPLACE WHERE`는 **조건에 맞는 기존 행을 삭제할 수 있습니다.**

| 상황 | 정책 |
|---|---|
| 원천에서 데이터가 **정말 없어졌을 수 있음**(삭제 반영이 필요) | `allow_empty_delete=True` → 비어 있으면 해당 월도 비움 |
| 조회 오류·일시 장애로 **0건처럼 보일 위험**이 있음 | `allow_empty_delete=False`(기본) → skip하고 기존 데이터 유지 |

대부분의 적재 배치는 후자가 안전합니다. 0건 월이 많이 나오면 경고를 모니터링하세요.

## 6-4. 부분 성공을 다운스트림이 허용하지 않을 때

"12개월이 모두 들어와야 의미가 있는" 경우에는 부분 반영이 오히려 문제일 수 있습니다.

- (간단) `failed_chunks` task value를 다운스트림이 확인해서, 실패가 있으면 후속 처리를 건너뜁니다.
- (엄격) 청크 결과를 **스테이징 테이블**(`run_id`, `chunk_id` 컬럼 포함)에 모았다가, 전부 성공했을 때만 L0에 반영합니다. 단계와 테이블이 하나 늘어납니다.

## 6-5. 동시 쓰기 충돌이 걱정될 때

공식 문서(Row-level concurrency) 기준 요점입니다.

- 테이블을 읽지 않는 순수 append(INSERT)끼리는 어떤 격리 수준에서도 **충돌하지 않습니다.**
- UPDATE/DELETE/MERGE는 **같은 파일 집합**을 건드릴 때만 충돌합니다. 문서는 "작업 조건에 쓰는 컬럼으로 파티셔닝하면 파일이 분리되어 충돌을 피할 수 있다"고 안내합니다.
- row-level concurrency는 **DBR 14.3 LTS 이상 + 파티션 없는 테이블 + deletion vectors 활성화**일 때 자동 적용되며, 같은 파일의 서로 다른 행을 건드리는 작업의 충돌을 자동 해결합니다. 파티션된 테이블에는 적용되지 않습니다.
- 문서의 충돌 표에는 `replaceWhere`가 별도 행으로 없습니다. **서로 다른 월의 `replaceWhere`를 동시에 실행했을 때의 충돌 여부는 문서에 명시돼 있지 않으니**, 실제 테이블에서 소규모로 테스트하세요. 참고로 공식 문서는 **liquid clustering 테이블이 row-level concurrency를 허용해 쓰기 충돌이 줄어든다**고 설명합니다.

충돌이 나면 다음 순서로 대응합니다.

1. 5-4의 재시도로 충돌한 청크를 다시 시도 (이미 `max_attempts`로 포함)
2. `max_workers`를 낮춤
3. 그래도 반복되면 **순차 실행**(`max_workers=1`) — 그래도 "한 달 실패해도 나머지 반영"이라는 목적은 유지됨

# 7. 주의할 점 체크리스트

- [ ] `replaceWhere` 조건과 **실제로 쓰는 데이터의 범위가 일치**해야 합니다. 어긋나면 오류로 중단됩니다.
- [ ] 청크 조건은 **Delta 컬럼으로 표현 가능**해야 합니다. Oracle `ROWID`, 시퀀스 범위처럼 Delta에 없는 값으로 분할하면 `replaceWhere`를 쓸 수 없습니다.
- [ ] Oracle `DATE`는 시간 정보를 포함합니다. 상한을 `<`(다음 달 1일)로 두면 안전합니다.
- [ ] 건수 확인은 `df.count()`(Oracle 재조회) 대신 **Delta에서 청크 범위만** 세세요.
- [ ] `max_workers` × (청크당 연결 수)가 Oracle의 허용 세션 수를 넘지 않게 하세요.
- [ ] 청크 쓰기 중 **스키마 변경**이 섞이면 다른 쓰기가 실패할 수 있습니다(메타데이터 변경은 동시 쓰기를 실패시킬 수 있음).
- [ ] 빈 청크 정책(6-3)을 의도적으로 정하세요.

# 8. 한 줄 정리

- 쓰기 한 번 = Delta 커밋 한 번이라서 한 task 실패가 전체 롤백이 되는 것은 **정상 동작**입니다.
- "읽을 때마다 저장"하려면 **청크(월)별로 읽기+쓰기를 독립 실행**해야 합니다.
- `DELETE` + `append`는 두 커밋이라 실패 시 빈 구간이 남습니다. **`replaceWhere` 한 번**으로 바꾸면 원자적이고 멱등합니다.
- 테이블 레이아웃은 공식 문서 권장대로 **Liquid clustering 우선**, 1 TB 이상이고 파티션이 충분히 클 때만 파티셔닝을 검토합니다.
- 서로 다른 월 `replaceWhere`의 동시 실행 충돌은 문서에 명시가 없으므로 **직접 테스트**합니다.

# 관련 용어 글
이 글에 나온 용어는 각각 별도 글로 자세히 정리했습니다.

Delta 트랜잭션(ACID) · 격리 수준 · Row-level concurrency · Deletion vectors · Delta 테이블 파티셔닝 · Liquid Clustering · JDBC 읽기 파티션 · replaceWhere · 청크(Chunk) · 멱등성 · For each task · Repair run · spark.task.maxFailures

## 🔗 참고
- [Isolation levels (WriteSerializable and Serializable)](https://docs.databricks.com/aws/en/optimizations/isolation/isolation-levels)
- [Row-level concurrency](https://docs.databricks.com/aws/en/optimizations/isolation/row-level-concurrency)
- [Selectively overwrite data with Delta Lake](https://docs.databricks.com/aws/en/delta/selective-overwrite)
- [Partitions](https://docs.databricks.com/aws/en/tables/partitions)
- [Use liquid clustering for tables](https://docs.databricks.com/aws/en/delta/clustering)
- [Use a For each task to run another task in a loop](https://docs.databricks.com/aws/en/jobs/tasks/for-each)
- [Troubleshoot and repair job failures](https://docs.databricks.com/aws/en/jobs/repair-job-failures)
- [PySpark DataFrameReader.jdbc](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrameReader.jdbc.html)
- [Spark JDBC data source](https://spark.apache.org/docs/latest/sql-data-sources-jdbc.html)
- [Spark configuration (spark.task.maxFailures)](https://spark.apache.org/docs/latest/configuration.html)
{% endraw %}
