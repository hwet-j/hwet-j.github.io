---
title:  "[DE/용어] 스키마 진화(Schema evolution, mergeSchema)"
excerpt: "들어오는 데이터에 맞춰 테이블 스키마를 자동으로 바꾸는 기능: 무엇을 해 주고 무엇은 해 주지 않는가"

categories:
  - DataEngineering
tags:
  - [Glossary, Delta Lake, Schema, Schema Evolution]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[Update table schemas with schema evolution](https://docs.databricks.com/aws/en/delta/update-schema)    
[Schema enforcement](https://docs.databricks.com/aws/en/tables/schema-enforcement)    
[Type widening](https://docs.databricks.com/aws/en/tables/features/type-widening)
{: .notice--warning}

# 0. 이 글을 읽기 전에

**스키마 강제(Schema enforcement)** 글에서 "Delta는 스키마가 안 맞으면 쓰기를 거부한다"를 설명했습니다. 이 글은 그 반대편, **"스키마를 필요에 따라 바꿔 가며 쓰고 싶을 때"** 를 다룹니다.

# 1. 한 줄 정의

**스키마 진화**는 데이터 요구사항이 바뀔 때 **테이블 구조를 변경하는 것** 전반을 뜻하고, 특히 **쓰기(INSERT/MERGE) 시점에 들어오는 데이터에 맞춰 테이블 스키마를 자동으로 바꾸는** 기능을 가리킵니다.

# 2. 어떤 변경을 지원하나 (공식 문서)

- 새 컬럼 **추가** (원하는 위치에)
- 기존 컬럼 **순서 변경**
- 기존 컬럼 **이름 변경** (컬럼 매핑 필요)
- 기존 컬럼의 **타입 넓히기** (type widening)

이 변경을 **명시적으로(DDL)** 하거나 **암묵적으로(DML 쓰기 중)** 할 수 있습니다.

> ⚠️ 공식 문서: **스키마 업데이트는 모든 동시 쓰기와 충돌**합니다. 변경을 조율하세요. 또한 스키마를 바꾸면 그 테이블을 읽던 **스트림이 종료**되어 다시 시작해야 합니다.
{: .notice--warning}

# 3. 명시적 변경 (ALTER TABLE)

데이터를 쓰지 않고 구조만 바꿉니다.

```sql
-- 컬럼 추가 (위치/설명 지정 가능)
ALTER TABLE main.l0.orders ADD COLUMNS (channel_cd STRING COMMENT '판매 채널');

-- 컬럼 이름 변경 (데이터 재작성 없이 하려면 column mapping 필요)
ALTER TABLE main.l0.orders RENAME COLUMN old_name TO new_name;

-- 컬럼 삭제 (메타데이터만 바꾸려면 column mapping 필요)
ALTER TABLE main.l0.orders DROP COLUMN legacy_col;
```

- 이름 변경·삭제를 **데이터 파일 재작성 없이** 하려면 테이블에 **column mapping**이 켜져 있어야 합니다.

# 4. 쓰기 중 자동 변경 (암묵적 스키마 진화)

## 4-1. 켜는 방법

| 방법 | 대상 | 비고 |
|---|---|---|
| `INSERT WITH SCHEMA EVOLUTION INTO ...` | SQL INSERT | DBR **18.1 이상** |
| `MERGE WITH SCHEMA EVOLUTION INTO ...` / `.withSchemaEvolution()` | MERGE | SQL 또는 API |
| `.option("mergeSchema", "true")` | 배치·스트리밍 DataFrame 쓰기 | 쓰기 작업 단위로 지정 |
| `spark.databricks.delta.schema.autoMerge.enabled = true` | SparkSession 전체 | **레거시** 방식 |

- 공식 문서는 **세션 전체 설정 대신, 쓰기마다 `WITH SCHEMA EVOLUTION` 또는 `mergeSchema`로 지정**하라고 권장합니다. 쓰기 단위의 옵션/구문이 **Spark 설정보다 우선**합니다.
- DBR 18.0 이하에서는 INSERT의 스키마 진화를 `mergeSchema` 옵션으로 켭니다.

## 4-2. 새 컬럼이 자동으로 추가된다

```python
# 소스에 channel_cd 컬럼이 있는데 대상 테이블에는 없는 경우
(df.write
   .option("mergeSchema", "true")
   .mode("append")
   .saveAsTable("main.l0.orders"))
```

- 소스에는 있고 대상에는 없는 컬럼이 **자동으로 대상 스키마에 추가**됩니다. 기존 행의 새 컬럼 값은 **NULL**입니다.
- **새 컬럼은 테이블 스키마의 맨 끝에 추가**됩니다. struct 안의 새 필드는 해당 struct의 끝에 추가됩니다.
- 컬럼 추가 시 **대소문자가 유지**됩니다.
- `WITH SCHEMA EVOLUTION`은 `INSERT INTO`, `INSERT OVERWRITE`, `INSERT INTO ... REPLACE` 형태를 지원하고, **대상은 Delta/Iceberg 테이블**이어야 합니다. (Hive 등 비-Delta 테이블에는 오류)

## 4-3. MERGE에서의 스키마 진화

MERGE는 소스와 대상 스키마 불일치를 다음 경우에 해결합니다.

- **소스에는 있고 대상에는 없는 컬럼**이 INSERT/UPDATE 액션에서 **이름으로 지정**되었거나, `UPDATE SET *` / `INSERT *`가 있을 때 → 그 컬럼이 대상에 추가되고 값이 채워집니다.
- 이는 **컬럼 이름과 구조가 소스와 대상 할당에서 정확히 일치**할 때만 적용됩니다. 액션에서 새 컬럼을 할당한다고 해서 컬럼이 "정의"되지는 않으며, **새 컬럼은 소스 스키마에 존재**해야 합니다.

```sql
-- newcol 이 소스에만 있으면 대상에 추가됨
UPDATE SET target.newcol = source.newcol
-- 소스에 있으나 대상에 없는 모든 컬럼/중첩 필드를 추가
UPDATE SET *
INSERT *
```

# 5. 스키마 진화가 "해 주지 않는 것" (오해 주의)

| 기대 | 실제 |
|---|---|
| "타입 불일치를 알아서 맞춰 준다" | 기본은 **아니다**. 타입 확장은 **type widening** 조건이 맞을 때만 |
| "테이블이 더 넓은 타입일 때 좁은 DataFrame을 받아 준다" | **스키마 진화의 대상이 아님**. 일반 스키마 강제 규칙을 따름 |
| "컬럼 삭제/이름 변경도 자동" | 자동 진화는 **컬럼 추가**가 중심. 이름 변경·삭제는 명시적 DDL |
| "mergeSchema를 켜면 안전하다" | 의도치 않은 **컬럼 추가**가 조용히 일어날 수 있음 |

## 타입 확장과의 관계 (공식 문서)

스키마 진화로 타입을 자동으로 넓히려면 다음을 **모두** 만족해야 합니다.

1. 쓰기가 **스키마 진화가 켜진 상태**로 실행
2. **대상 테이블에 type widening이 켜져** 있음
3. **소스 컬럼 타입이 대상 컬럼 타입보다 넓음**
4. type widening이 **그 변경을 지원**

이 조건에 맞지 않는 타입 불일치는 **일반 스키마 강제 규칙**을 따릅니다. 그리고 **type widening이 꺼져 있으면, 스키마 진화는 대상 테이블 타입에 맞게 downcast를 시도**합니다.

→ 예를 들어 **대상이 BIGINT, 소스가 INT**인 경우는 조건 3("소스가 더 넓다")을 만족하지 않아서 이 경로로 해결되지 않습니다. (→ INT DataFrame을 BIGINT 테이블에 쓸 때 사례 글)

# 6. 언제 켜고, 언제 끄는가

| 상황 | 권장 |
|---|---|
| 원천 스키마가 **자주 바뀌는** 수집 파이프라인(JSON 로그 등) | 쓰기 단위로 `mergeSchema` 사용 고려 |
| **엄격한 스키마**가 중요한 정제/마트 테이블 | 켜지 않고 **스키마 강제**로 막는다 |
| 일회성 마이그레이션 | 명시적 `ALTER TABLE`로 변경 |
| 여러 Job이 같은 테이블에 동시에 쓴다 | 스키마 변경은 **모든 동시 쓰기와 충돌**하므로 조율 |

# 7. 타입이나 이름을 임의로 바꾸려면: `overwriteSchema`

타입 변경·이름 변경·컬럼 삭제를 **테이블을 다시 쓰면서** 하려면 `overwriteSchema`를 사용합니다. (공식 문서 예시)

```python
(spark.read.table("main.l0.orders")
   .withColumn("order_dt", F.col("order_dt").cast("date"))
   .write
   .mode("overwrite")
   .option("overwriteSchema", "true")
   .saveAsTable("main.l0.orders"))
```

- **전체 데이터를 다시 쓰므로** 큰 테이블에서는 비용이 큽니다. 작은 테이블(예: 이력·코드 테이블)에서 현실적입니다.

# 8. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "`mergeSchema`는 만능 스키마 맞춤 옵션" | 주로 **새 컬럼 추가**. 타입은 조건부 |
| "세션 설정으로 켜 두면 편하다" | 공식 문서는 **쓰기 단위 지정을 권장**, 세션 설정은 레거시 |
| "스키마를 바꾸면 기존 작업은 그대로 돈다" | 동시 쓰기 **충돌**, 읽던 **스트림 종료** |

# 9. 한 줄 정리

- 스키마 진화 = **쓰기 중 또는 DDL로 테이블 스키마를 바꾸는 것**. 쓰기 중 자동 변경은 `mergeSchema` / `WITH SCHEMA EVOLUTION`.
- 자동 변경의 중심은 **새 컬럼 추가**이며, 새 컬럼은 **끝에 추가**되고 기존 행은 **NULL**.
- **타입 확장은 type widening 조건**(대상 켜짐 + 소스가 더 넓음)이 맞아야 하고, 그렇지 않으면 **스키마 강제 규칙**을 따른다.
- 스키마 변경은 **동시 쓰기와 충돌**하고 **스트림을 종료**시킨다.

# 관련 용어
스키마 강제(Schema enforcement), Type widening, Delta 프로토콜과 Table feature

## 🔗 참고
- [Update table schemas with schema evolution](https://docs.databricks.com/aws/en/delta/update-schema)
- [Schema enforcement](https://docs.databricks.com/aws/en/tables/schema-enforcement)
- [Type widening](https://docs.databricks.com/aws/en/tables/features/type-widening)
