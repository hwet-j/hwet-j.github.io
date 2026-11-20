---
title:  "[DE/Databricks] INT DataFrame을 BIGINT 테이블에 쓸 때: SQL INSERT는 되는데 saveAsTable append는 실패한다"
excerpt: "DELTA_FAILED_TO_MERGE_FIELDS의 원인인 INT/BIGINT 타입 불일치, 쓰기 방법별 차이와 해결 방법"

categories:
  - DataEngineering
tags:
  - [Databricks, Delta Lake, Schema Enforcement, saveAsTable, cast]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

업무에서 겪은 상황을 일반화해서 정리했습니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서(AWS 기준)를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[Schema enforcement](https://docs.databricks.com/aws/en/tables/schema-enforcement) ·
[Update table schemas](https://docs.databricks.com/aws/en/delta/update-schema) ·
[Type widening](https://docs.databricks.com/aws/en/tables/features/type-widening)
{: .notice--warning}

# 1. 한눈에 보는 요약

| 항목 | 내용 |
|---|---|
| 현상 | `INT` 컬럼을 가진 DataFrame을 `BIGINT` 컬럼의 Delta 테이블에 `saveAsTable`(`append`)로 쓰면 `DELTA_FAILED_TO_MERGE_FIELDS` 오류로 **실패** |
| 같은 상황에서 | **SQL `INSERT`는 성공**하는 것으로 공식 문서 예시에 나와 있음 (정수 `42` → `BIGINT` 컬럼) |
| 원인 | **값이 아니라 타입**의 불일치. 값이 INT 범위 안이어도 실패 |
| 해결 | DataFrame 컬럼을 **대상 테이블의 실제 타입으로 `cast`** 한 뒤 쓰기 |

# 2. 상황

Delta 테이블의 숫자 컬럼은 `BIGINT`인데, 쓰려는 DataFrame의 같은 컬럼은 `INT`입니다.

```
대상 테이블   main.demo.target_tbl ( id STRING, row_cnt BIGINT )
쓸 DataFrame  ( id STRING, row_cnt INT )          ← 파이썬 정수를 lit() 으로 넣어 만든 경우 등
```

```python
df.write.mode("append").saveAsTable("main.demo.target_tbl")
```

```
[DELTA_FAILED_TO_MERGE_FIELDS] Failed to merge fields 'ROW_CNT' and 'ROW_CNT'
SQLSTATE: 22005
```

- 메시지에는 **두 필드의 타입이 표시되지 않습니다.** "같은 이름의 필드를 병합하지 못했다"만 알려 줍니다.
- 같은 오류가 **값이 `0`이든 `10억`이든** 발생합니다. 값의 크기와 무관합니다.

# 3. 쓰기 방법별 결과 정리

같은 "INT 데이터를 BIGINT 컬럼에 쓴다"인데 **쓰기 방법에 따라 결과가 갈립니다.**

| 쓰기 방법 | 결과 | 근거 |
|---|---|---|
| **SQL `INSERT INTO ... VALUES / SELECT`** | ✅ 성공 (안전하게 cast) | **공식 문서** 예시 (정수 `42` → `BIGINT`) |
| `MERGE INTO` | 안전하게 cast를 시도 | **공식 문서** |
| **DataFrame `saveAsTable` + `mode("append")`** | ❌ **`DELTA_FAILED_TO_MERGE_FIELDS`** | **직접 재현** (임시 테이블) |
| DataFrame `insertInto` | ✅ 성공으로 보고됨 | Delta GitHub 이슈 (공식 아님, **직접 확인 필요**) |
| `saveAsTable` + `mergeSchema` | ❌ INT를 BIGINT로 확장하지 않음 | 커뮤니티 답변 (공식 아님, **직접 확인 필요**) |

> 공식 문서는 INSERT/MERGE에서 **"타입을 안전하게 cast하려고 시도한다"** 고 설명합니다. 반면 **DataFrame `saveAsTable` append의 INT→BIGINT 동작은 공식 문서에 직접 나와 있지 않습니다.** 그래서 위 표의 ❌ 행은 문서가 아니라 **재현 실험**에 근거합니다.
{: .notice--info}

# 4. 직접 재현하고 비교하기

한 번에 모든 방법을 실행해 보는 스크립트입니다. 결과는 환경(런타임 버전)에 따라 다를 수 있으니 **직접 실행해서 확인**하세요.

```python
tbl = "main.demo.type_check"
spark.sql(f"CREATE OR REPLACE TABLE {tbl} (id STRING, row_cnt BIGINT)")

df_int = spark.createDataFrame([("a", 1)], "id STRING, row_cnt INT")
df_int.createOrReplaceTempView("v_int")

def attempt(name, fn):
    try:
        fn(); print(f"[OK ] {name}")
    except Exception as e:
        print(f"[ERR] {name}: {str(e)[:120]}")

attempt("① saveAsTable append",          lambda: df_int.write.mode("append").saveAsTable(tbl))
attempt("② insertInto",                  lambda: df_int.write.insertInto(tbl))
attempt("③ SQL INSERT ... SELECT",       lambda: spark.sql(f"INSERT INTO {tbl} (id, row_cnt) SELECT id, row_cnt FROM v_int"))
attempt("④ SQL INSERT ... VALUES",       lambda: spark.sql(f"INSERT INTO {tbl} VALUES ('b', 2)"))
attempt("⑤ saveAsTable + mergeSchema",   lambda: df_int.write.mode("append").option("mergeSchema","true").saveAsTable(tbl))
attempt("⑥ cast 후 saveAsTable append",  lambda: df_int.withColumn("row_cnt", df_int.row_cnt.cast("bigint"))
                                                   .write.mode("append").saveAsTable(tbl))

spark.sql(f"DROP TABLE IF EXISTS {tbl}")     # 정리
```

- 이 환경에서 확인된 것은 **① 실패, ⑥ 성공**입니다. ②~⑤는 위 표의 근거(공식 문서/이슈/커뮤니티)와 같은지 직접 실행해 확인하세요.
- `②`의 `insertInto`는 **컬럼을 이름이 아니라 위치 기준**으로 맞추는 것으로 알려져 있으므로, 쓸 때 컬럼 순서를 반드시 확인하세요(직접 확인 권장).

# 5. 왜 쓰기 방법마다 다를까

공식 문서에는 이 차이를 설명하는 부분이 없습니다. 다음은 **단서에 기반한 추정**입니다.

- SQL `INSERT`는 "대상 컬럼 타입에 맞게 **안전하게 cast**한다"는 규칙이 문서에 명시되어 있고, 이 단계에서 타입이 맞춰지는 것으로 보입니다.
- DataFrame `saveAsTable` append는 오류 이름이 **`DELTA_FAILED_TO_MERGE_FIELDS`("필드 병합 실패")** 인 점으로 보아, **DataFrame 스키마와 테이블 스키마를 병합(merge)하는 단계에서 같은 이름의 필드 타입이 다르면 실패**하는 것으로 보입니다.

> 위 설명은 **오류 이름과 재현 결과에서 추정한 것**이며 내부 동작을 공식 문서가 확인해 주지는 않습니다. 근거 없이 단정하지 말고, 중요한 파이프라인에서는 **직접 테스트**로 동작을 확인하세요.
{: .notice--warning}

# 6. 방향 정리: type widening과 혼동하지 말 것

```
[이번 사례]      테이블 BIGINT(넓음) ← DataFrame INT(좁음)    → 불일치로 실패
[type widening]  테이블 INT(좁음)    ← 데이터 BIGINT(넓음)    → 테이블 타입을 BIGINT로 "넓혀 주는" 기능
```

- Type widening은 **들어오는 데이터가 더 넓을 때 테이블 타입을 넓혀 주는** 기능입니다. **테이블이 이미 더 넓은 이번 사례와는 방향이 반대**입니다. (→ Type widening 글)
- 공식 문서: 스키마 진화로 타입을 자동으로 넓히려면 **① 스키마 진화가 켜져 있고 ② 대상 테이블에 type widening이 켜져 있고 ③ 소스 컬럼 타입이 대상보다 넓고 ④ type widening이 그 변경을 지원**해야 합니다. 이 조건에 맞지 않는 타입 불일치는 **일반 schema enforcement 규칙**을 따릅니다.
- 그리고 **type widening이 꺼져 있으면 스키마 진화는 대상 타입에 맞게 downcast를 시도**합니다.
- 따라서 이번 문제는 **`mergeSchema`나 type widening 옵션이 빠져서 생긴 문제가 아닙니다.**

## 조합별 정리

| 테이블 | DataFrame | 방향 | 결과 |
|---|---|---|---|
| BIGINT | **INT** | 데이터가 **좁음** | `saveAsTable` append **실패** (이번 사례) |
| INT | BIGINT | 데이터가 **넓음** | 스키마 진화 + type widening 조건이 맞으면 테이블이 **넓혀짐**, 아니면 일반 규칙/downcast 시도 |
| BIGINT | BIGINT | 동일 | 성공 |
| INT | INT | 동일 | 성공 |

# 7. 흔한 원인: INT/BIGINT는 이렇게 섞여 들어온다

"나는 BIGINT를 넣은 것 같은데 왜 INT지?"가 되는 경우가 많습니다. 아래는 PySpark에서 **알려진 동작**이며, 반드시 `printSchema()`로 직접 확인하세요(추정).

| 만드는 방법 | 흔히 추론되는 타입 |
|---|---|
| `lit(5)`처럼 **작은 파이썬 정수 리터럴** | **INT** |
| `spark.createDataFrame([(1,)])`처럼 파이썬 값으로 DataFrame 생성 시 정수 추론 | **BIGINT** |
| `spark.range(n)`의 `id` | **BIGINT** |
| `count()` 결과 | **BIGINT** |
| 파이썬 `len(...)`, 계산한 정수를 `lit()`에 전달 | 값 범위에 따라 INT 또는 BIGINT (큰 값은 BIGINT로 추론되는 것으로 알려짐) |

```python
from pyspark.sql.functions import lit
spark.range(1).select(
    lit(5).alias("lit_small"),
    lit(2**40).alias("lit_large"),
).printSchema()
spark.createDataFrame([(1,)], ["x"]).printSchema()
spark.range(1).printSchema()
```

> 같은 파이프라인에서도 **값이 작을 땐 INT, 커지면 BIGINT**로 추론이 바뀌어, 어제까지 되던 쓰기가 갑자기 실패할 수 있습니다. 그래서 **추론에 의존하지 말고 명시적으로 cast** 하는 편이 안전합니다.
{: .notice--info}

# 8. 해결

## 8-1. 변경 전/후 코드 (컬럼 하나)

```python
# 변경 전: 타입을 지정하지 않음 → 파이썬 int 가 IntegerType 으로 들어감
df = spark.range(1).withColumn("row_cnt", lit(total_rows))
df.write.mode("append").saveAsTable("main.demo.target_tbl")          # 대상 BIGINT 면 실패
```

```python
# 변경 후: 대상 테이블의 '실제' 컬럼 타입으로 cast
target_type = spark.table("main.demo.target_tbl").schema["row_cnt"].dataType
df = spark.range(1).withColumn("row_cnt", lit(total_rows).cast(target_type))
df.write.mode("append").saveAsTable("main.demo.target_tbl")
```

- **`cast("bigint")`로 고정하지 않고 대상 테이블의 타입을 읽어 맞추는** 이유: 같은 코드가 `INT` 테이블과 `BIGINT` 테이블에 모두 쓰일 때, 고정하면 **반대쪽 테이블이 깨집니다.**

## 8-2. DataFrame 전체를 테이블 스키마에 맞추기

```python
from pyspark.sql import functions as F

def align_to_table_schema(spark, df, table):
    """DataFrame 컬럼을 대상 테이블의 컬럼 타입으로 cast (이름 기준, 테이블 컬럼 순서로 정렬)."""
    target = spark.table(table).schema
    missing = [f.name for f in target.fields if f.name not in df.columns]
    if missing:
        raise ValueError(f"DataFrame 에 없는 컬럼: {missing}")
    return df.select([F.col(f.name).cast(f.dataType).alias(f.name) for f in target.fields])

align_to_table_schema(spark, df, "main.demo.target_tbl") \
    .write.mode("append").saveAsTable("main.demo.target_tbl")
```

- 컬럼마다 cast를 빠뜨릴 일이 없고, 컬럼 순서도 테이블 기준으로 맞춰집니다.

## 8-3. 더 좁은 타입으로 cast할 때: 값 범위 검증

대상이 **INT**인데 값이 BIGINT로 들어올 때(좁히는 방향)는 **오버플로**가 날 수 있습니다. 오버플로 시 동작은 ANSI 설정 등에 따라 다를 수 있으니, 명시적으로 검증하세요.

```python
INT_MAX = 2_147_483_647

def check_fits_int(value):
    if value > INT_MAX or value < -INT_MAX - 1:
        raise ValueError(f"값 {value} 가 INT 범위를 초과합니다. 대상 컬럼을 BIGINT로 변경하세요.")
```

# 9. 상황별 선택

| 방법 | 설명 | 적합한 경우 | 주의 |
|---|---|---|---|
| **A. 대상 타입으로 cast 후 `saveAsTable`** | 8장 방식 | 가장 일반적, 즉시 적용 | 좁히는 cast는 오버플로 검증 |
| **B. SQL `INSERT`로 쓰기** (`createOrReplaceTempView` → `INSERT INTO ... SELECT`) | 공식 문서상 safe cast 시도 | DataFrame 코드를 바꾸기 어려울 때 | **컬럼 목록을 명시**(`INSERT INTO t (c1, c2) SELECT ...`) |
| C. `insertInto` | 이슈상 성공 보고 | 간단히 쓰고 싶을 때 | **위치 기준**일 수 있어 컬럼 순서 확인, 직접 테스트 필요 |
| **D. 대상 DDL의 타입 통일** | 신규 테이블은 처음부터 BIGINT | 건수·ID처럼 커질 수 있는 컬럼 | `CREATE TABLE IF NOT EXISTS`는 **기존 테이블에 영향 없음** |
| E. Type widening | **테이블이 INT이고** BIGINT로 넓히고 싶을 때 | 기존 INT 테이블을 BIGINT로 | **DBR 15.4 LTS+**, 프로토콜 상승, 읽기 호환성 제한 (→ Delta 프로토콜 글) |
| F. `overwriteSchema` | 테이블을 다시 쓰며 타입 변경 | 작은 테이블 | 전체 재작성 |
| ✗ `mergeSchema` | — | **이번 방향에는 효과 없음** | 6장 |

## 9-1. SQL INSERT 경유 예시

```python
df.createOrReplaceTempView("v_src")
spark.sql("""
    INSERT INTO main.demo.target_tbl (id, row_cnt)
    SELECT id, row_cnt FROM v_src
""")
```

## 9-2. 기존 INT 테이블을 BIGINT로 넓히기 (type widening)

```sql
ALTER TABLE main.demo.target_tbl SET TBLPROPERTIES ('delta.enableTypeWidening' = 'true');
ALTER TABLE main.demo.target_tbl ALTER COLUMN row_cnt TYPE BIGINT;
```

- 공식 문서: DBR **15.4 LTS 이상**, 활성화하면 **reader/writer 프로토콜이 올라가며** type widening이 켜진 테이블은 **DBR 15.4 LTS 이상에서만 읽을 수 있습니다.** 읽는 도구와 클러스터 버전을 먼저 확인하세요.

# 10. 이 글의 근거 (무엇을 직접 테스트했고, 무엇이 문서·추정인가)

| # | 주장 | 근거 유형 | 근거 내용 |
|---|---|---|---|
| 1 | INT DataFrame을 BIGINT 테이블에 `saveAsTable` append하면 `DELTA_FAILED_TO_MERGE_FIELDS`로 실패한다 | **직접 테스트** | 운영 적재 Job에서 실제 발생 → **임시 테이블(`BIGINT` 컬럼)에 INT DataFrame을 append해 같은 오류를 재현** |
| 2 | 값의 크기와 무관하다 | **직접 확인(실제 로그)** | 같은 Job에서 SUCCESS 이력(건수 약 14억)과 `except`의 FAIL 이력(건수 `0`) **둘 다 같은 오류** |
| 3 | 오버플로가 원인이 아니다 | **직접 확인(실측)** | 적재 건수가 INT 최댓값(2,147,483,647) **이내** |
| 4 | 건수 컬럼을 대상 테이블 타입으로 cast하면 해결된다 | **직접 테스트** | cast 적용 후 **같은 오류가 발생하지 않음**을 확인 |
| 5 | 건수 컬럼에 cast가 없었고 파이썬 int가 `IntegerType`이 되었다 | **직접 코드 확인** | 해당 컬럼만 `lit(값)`에 cast가 없었고 다른 컬럼에는 cast가 있었음 |
| 6 | SQL `INSERT`는 정수를 BIGINT 컬럼에 안전하게 cast한다 | **공식 문서** | Schema enforcement 문서의 `INSERT` 예시(`42` → `BIGINT`). **이 환경에서 직접 테스트하지는 않음** |
| 7 | type widening은 이번 사례와 방향이 반대다 | **공식 문서** | Type widening 문서(대상이 좁고 데이터가 넓을 때 + 대상에 기능이 켜져 있을 때) |
| 8 | `insertInto`는 성공하고 `mergeSchema`는 INT를 확장하지 않는다 | **공식 아님(외부 보고)** | Delta Lake 이슈, 커뮤니티 답변. **직접 확인 필요** |
| 9 | 작은 정수 `lit()`은 INT, 큰 값은 BIGINT로 추론된다 | **추정** | 알려진 PySpark 동작이나 **직접 확인 필요**(`printSchema()`) |
| 10 | 오류 이름이 "필드 병합 실패"이므로 스키마 병합 단계에서 실패하는 것 같다 | **추정** | 오류 이름과 재현 결과에서의 추정. 내부 동작은 문서에 없음 |

- **테스트 환경**: Databricks on AWS, Unity Catalog. (Databricks Runtime 버전은 기록되어 있지 않음 → 재현 시 **DBR 버전을 함께 기록**해 두세요.)
- 이 글의 **결론 중 직접 테스트로 확인된 것은 1~5번**이고, 나머지는 문서나 추정입니다. 쓰기 방법별 결과(4장 표)는 직접 실행해서 확인하는 것을 권합니다.
- 이 사례는 포트폴리오 자료로도 정리했습니다. (`portfolio/` 폴더)

# 11. 체크리스트

- [ ] `DELTA_FAILED_TO_MERGE_FIELDS`를 보면 **컬럼 타입**(INT/BIGINT, DECIMAL 정밀도, TIMESTAMP/TIMESTAMP_NTZ)부터 비교하는가? (오류 메시지에 타입이 안 나옴)
- [ ] 쓰기 전에 **DataFrame 스키마와 테이블 스키마를 나란히 출력**해 비교하는가?
- [ ] 숫자를 **추론에 맡기지 않고 명시적으로 cast**하는가?
- [ ] 같은 코드가 **여러 테이블**(INT/BIGINT 혼재)에 쓰인다면 **대상 타입을 읽어서 맞추는가**?
- [ ] 좁히는 cast에는 **값 범위 검증**이 있는가?
- [ ] 신규 테이블 DDL에서 건수·ID 같은 컬럼을 **BIGINT**로 정했는가?

## 빠른 진단 코드

```python
src = {f.name: f.dataType for f in df.schema.fields}
for f in spark.table("main.demo.target_tbl").schema.fields:
    if src.get(f.name) != f.dataType:
        print(f"[타입 불일치] {f.name}: DataFrame={src.get(f.name)}, Table={f.dataType}")
```

# 12. 한 줄 정리

- **INT DataFrame → BIGINT 테이블**을 `saveAsTable` append로 쓰면 **값과 무관하게 타입 불일치로 실패**한다. (재현 확인)
- 공식 문서는 **SQL INSERT / MERGE가 safe cast를 시도**한다고 설명하지만, **DataFrame `saveAsTable` append의 이 동작은 문서에 직접 없다.** 쓰기 방법별 차이는 직접 테스트로 확인한다.
- **type widening / `mergeSchema`는 방향이 반대**라서 이 문제의 해결책이 아니다.
- 해결은 **대상 테이블의 실제 타입으로 cast**(고정 `bigint` 금지), 필요하면 SQL INSERT 경유, 신규 DDL은 BIGINT로.
- INT/BIGINT는 **추론에 따라 섞여 들어오므로** 명시적 cast와 스키마 비교 습관이 안전하다.

# 관련 용어 글
스키마 강제(Schema enforcement) · 스키마 진화(Schema evolution, mergeSchema) · Type widening · Delta 프로토콜과 Table feature

## 🔗 참고
- [Schema enforcement](https://docs.databricks.com/aws/en/tables/schema-enforcement)
- [Update table schemas with schema evolution](https://docs.databricks.com/aws/en/delta/update-schema)
- [Type widening](https://docs.databricks.com/aws/en/tables/features/type-widening)
- (공식 아님) [Delta Lake 이슈 #2988](https://www.github.com/delta-io/delta/issues/2988)
- (공식 아님) [Databricks 커뮤니티: mergeSchema와 INT/BIGINT](https://community.databricks.com/t5/data-engineering/can-t-mergeschema-handle-int-and-bigint/m-p/135518)
