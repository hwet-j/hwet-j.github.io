---
title:  "[DE/용어] 스키마 강제(Schema enforcement)"
excerpt: "Delta 테이블이 '맞지 않는 데이터'를 쓰기 시점에 막는 규칙과, INSERT·MERGE·DataFrame append의 차이"

categories:
  - DataEngineering
tags:
  - [Glossary, Delta Lake, Schema, Schema Enforcement]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[Schema enforcement](https://docs.databricks.com/aws/en/tables/schema-enforcement)    
[Update table schemas with schema evolution](https://docs.databricks.com/aws/en/delta/update-schema)
{: .notice--warning}

# 0. 이 글을 읽기 전에

"왜 데이터를 쓰는데 `DELTA_FAILED_TO_MERGE_FIELDS` 같은 오류가 나지?"의 배경 개념입니다. **스키마(schema)** 는 테이블의 **컬럼 이름과 타입의 약속**입니다. 예를 들어 `order_id STRING, amount DECIMAL(18,2)`처럼요. Delta 테이블은 이 약속을 **쓰기 시점에 검사**합니다.

# 1. 한 줄 정의

**스키마 강제(Schema enforcement)** 는 Delta 테이블에 데이터를 쓸 때, **테이블 스키마와 맞지 않는 데이터를 거부해서 데이터 품질을 지키는** 기능입니다. 공식 문서: "Databricks는 Delta Lake 테이블에 대해 **쓰기 시점에 스키마를 강제**한다." (CSV, JSON 같은 비-Delta 형식에는 적용되지 않음)

# 2. 왜 필요한가

스키마를 검사하지 않으면 다음과 같은 사고가 납니다.

| 상황 | 검사 없을 때 | 스키마 강제가 있을 때 |
|---|---|---|
| 오타로 `amuont` 컬럼을 써 넣음 | 이상한 컬럼이 조용히 생기거나 값이 NULL | 쓰기 **실패**, 존재하는 컬럼명을 제안 |
| 숫자 컬럼에 문자열을 넣음 | 쓰레기 값 적재 | 타입 불일치 **실패** |
| 컬럼 순서가 바뀐 파일을 적재 | 값이 엉뚱한 컬럼에 들어감 | (규칙에 따라) 이름 기준 검사 |

"일단 넣고 나중에 정리"가 아니라 **"안 맞으면 처음부터 막는다"** 는 철학입니다.

# 3. 규칙 (공식 문서)

## 3-1. INSERT를 할 때

1. **넣는 모든 컬럼은 대상 테이블에 존재**해야 한다.
2. **모든 컬럼의 데이터 타입이 대상 테이블과 일치**해야 한다.
3. 단, Databricks는 **타입을 대상 테이블에 맞게 "안전하게 cast하려고 시도"** 한다.

```sql
CREATE TABLE main.default.enforce_demo (id INT, name STRING, amount BIGINT);

-- ❌ unknown_column 이 테이블에 없음 → UNRESOLVED_COLUMN.WITH_SUGGESTION (SQLSTATE 42703), 유효한 컬럼명을 제안
INSERT INTO main.default.enforce_demo (id, unknown_column) VALUES (1, 'value');

-- ✅ 정수 42를 BIGINT 컬럼 amount 로 안전하게 cast 하여 성공
INSERT INTO main.default.enforce_demo (id, amount) VALUES (1, 42);
```

## 3-2. MERGE를 할 때

1. 소스 타입이 대상 컬럼과 다르면 **안전하게 cast를 시도**한다.
2. UPDATE/INSERT 액션의 **대상 컬럼은 대상 테이블에 존재**해야 한다. (없으면 `DELTA_MERGE_UNRESOLVED_EXPRESSION`)
3. `INSERT *` / `UPDATE SET *`를 쓸 때는
   - **소스에 대상 테이블의 모든 컬럼이 있어야** 하고,
   - 소스에만 있는 **추가 컬럼은 무시**한다 (예: 소스의 `extra_col`).

# 4. "안전하게 cast"란 무엇인가

값이 **손실 없이** 대상 타입으로 바뀔 수 있을 때를 뜻하는 것으로 이해하면 됩니다. (개념 설명)

| 방향 | 예 | 일반적인 성격 |
|---|---|---|
| 좁은 → 넓은 | INT → BIGINT, INT → DECIMAL(18,2) | 값 손실 없음 (안전) |
| 넓은 → 좁은 | BIGINT → INT, DECIMAL(18,2) → DECIMAL(10,2) | **오버플로/잘림 위험** |
| 문자열 → 숫자 | `'123'` → INT | 값에 따라 실패 가능 |

> 어떤 변환을 "안전"하다고 보는지의 정확한 규칙(허용 가능한 조합, ANSI 설정에 따른 차이)은 이번에 공식 문서에서 상세하게 확인하지 못했습니다. 중요한 컬럼은 **직접 테스트**하세요.
{: .notice--info}

# 5. 쓰는 방법에 따라 달라 보이는 동작

실무에서 헷갈리는 부분입니다.

| 쓰기 방법 | 타입이 다를 때 | 근거 |
|---|---|---|
| SQL `INSERT` | 안전한 cast를 시도해서 대체로 성공 (예: 42 → BIGINT) | **공식 문서 예시** |
| `MERGE` | 안전한 cast를 시도 | **공식 문서** |
| DataFrame `saveAsTable`(`mode("append")`) | 컬럼 타입이 다르면(예: DataFrame INT, 테이블 BIGINT) **`DELTA_FAILED_TO_MERGE_FIELDS`로 실패하는 사례** | **직접 재현**, 커뮤니티/이슈 보고. 공식 문서에 직접 설명 없음 |

- 같은 "타입이 다른 데이터를 쓴다"인데도 **쓰기 API에 따라 동작이 다를 수 있다**는 점이 핵심입니다. (→ INT DataFrame을 BIGINT 테이블에 쓸 때 사례 글)
- DataFrame API로 쓸 때는 **쓰기 전에 DataFrame을 테이블 스키마에 맞추는 습관**이 안전합니다.

```python
target = spark.table("main.l0.orders").schema
df = df.select([F.col(f.name).cast(f.dataType) for f in target.fields])   # 이름 기준으로 타입 맞춤
df.write.mode("append").saveAsTable("main.l0.orders")
```

# 6. 스키마를 바꾸고 싶을 때

스키마 강제는 "마음대로 바뀌지 않게" 막는 장치이므로, **의도적으로** 바꾸는 방법이 따로 있습니다.

| 목적 | 방법 |
|---|---|
| 컬럼 추가/순서/이름 변경 등 **명시적** 변경 | `ALTER TABLE` |
| 들어오는 데이터에 맞춰 **자동 변경** | 스키마 진화(`mergeSchema`, `WITH SCHEMA EVOLUTION`) (→ 스키마 진화 글) |
| 컬럼 타입을 **넓히기** | type widening (→ Type widening 글) |
| 컬럼 타입을 **좁히거나 임의 변경** | 테이블을 다시 쓰는 `overwriteSchema` |

> 스키마 변경은 **동시에 진행 중인 모든 쓰기와 충돌**하고, 해당 테이블을 읽고 있던 **스트림을 종료**시킵니다. 변경 시점을 조율하세요. (공식 문서)
{: .notice--warning}

# 7. 외부 테이블 주의

외부 클라이언트로 외부 테이블의 메타데이터를 직접 바꾸거나 경로 기반으로 접근하면, Unity Catalog가 스키마 변경을 **자동으로 동기화하지 않아** 스키마 강제가 제대로 적용되지 않을 수 있습니다. 이때는 `MSCK REPAIR TABLE <table-name> SYNC METADATA`로 동기화합니다.

# 8. 오류를 만났을 때 보는 순서

1. **컬럼 이름**이 테이블에 있는가? (오타, 대소문자)
2. **컬럼 타입**이 같은가? (INT/BIGINT, DECIMAL 정밀도, TIMESTAMP/TIMESTAMP_NTZ)
3. **컬럼 개수/순서**가 의도와 같은가?
4. 쓰기 방법이 **INSERT/MERGE/DataFrame** 중 무엇인가? (동작이 다를 수 있음)

```python
df.printSchema()
spark.table("main.l0.orders").printSchema()
```

> `DELTA_FAILED_TO_MERGE_FIELDS` 같은 오류 메시지에는 **두 필드의 타입이 표시되지 않을 수 있으므로** 직접 두 스키마를 나란히 비교해야 합니다.

# 9. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "타입이 달라도 값이 들어가면 되지" | Delta는 **타입 불일치를 거부**(또는 안전한 cast만 허용) |
| "INSERT에서 되면 DataFrame append에서도 된다" | **쓰기 API에 따라 다를 수 있음** (재현 사례 있음) |
| "`mergeSchema`를 켜면 타입 불일치도 해결" | 주로 **새 컬럼 추가**용. 타입은 type widening 조건이 맞을 때만 |
| "CSV/JSON도 같은 검사를 한다" | 비-Delta 형식에는 스키마 강제가 적용되지 않음 |

# 10. 한 줄 정리

- 스키마 강제 = **Delta가 쓰기 시점에 스키마가 안 맞는 데이터를 거부**하는 기능.
- 규칙: **없는 컬럼 불가, 타입 일치(안전한 cast는 시도)**. MERGE의 `INSERT *`는 **추가 컬럼은 무시**, 필요한 컬럼이 빠지면 오류.
- **쓰기 API(INSERT / MERGE / DataFrame append)에 따라 타입 처리가 달라 보일 수 있다.** DataFrame은 **미리 테이블 스키마에 맞춰 cast**하는 것이 안전하다.
- 스키마를 바꾸려면 `ALTER TABLE`, 스키마 진화, type widening, `overwriteSchema` 중 목적에 맞게 선택한다.

# 관련 용어
스키마 진화(Schema evolution), Type widening, Delta 프로토콜과 Table feature

## 🔗 참고
- [Schema enforcement](https://docs.databricks.com/aws/en/tables/schema-enforcement)
- [Update table schemas with schema evolution](https://docs.databricks.com/aws/en/delta/update-schema)
