---
title:  "[DE/용어] replaceWhere (선택적 덮어쓰기)"
excerpt: "조건에 맞는 행만 원자적으로 교체하는 방법과 REPLACE USING/ON과의 차이"

categories:
  - DataEngineering
tags:
  - [Glossary, Delta Lake, replaceWhere, Overwrite]

toc: true
toc_sticky: true
published: true

date: 2026-11-19
last_modified_at: 2026-11-19
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-19)    
[Selectively overwrite data with Delta Lake](https://docs.databricks.com/aws/en/delta/selective-overwrite)
{: .notice--warning}

# 0. 이 글을 읽기 전에

"기존 데이터를 지우고 다시 적재한다"를 **안전하게** 하는 방법을 다룹니다. **Delta 트랜잭션(ACID)** 글의 "SQL 문 하나 = 트랜잭션 하나"를 먼저 보시면 이 글의 필요성이 바로 이해됩니다.

# 1. 한 줄 정의

**조건(predicate)에 맞는 행만 원자적으로 교체**하는 Delta 쓰기 방식입니다. "해당 범위 삭제 + 새 데이터 삽입"을 **한 번의 커밋**으로 처리합니다.

# 2. 왜 필요한가: DELETE + append의 위험

같은 범위를 "지우고 다시 넣는" 가장 단순한 방법은 두 문장을 이어서 실행하는 것입니다.

```sql
DELETE FROM main.l0.orders WHERE order_dt >= '2026-01-01' AND order_dt < '2026-02-01';   -- 커밋 ①
INSERT INTO main.l0.orders SELECT ... ;                                                    -- 커밋 ②
```

```
시간 ─────────────────────────────────────────────►
①  DELETE 커밋 완료   → 1월 데이터 사라짐
②  INSERT 실행 중 실패 → 1월이 비어 있는 채로 남음  ❌
```

`replaceWhere`는 이를 **하나의 원자적 쓰기**로 바꿉니다.

```
replaceWhere:  [1월 삭제 + 새 1월 삽입] = 커밋 1번
               ├─ 성공 → 한꺼번에 교체됨
               └─ 실패 → 아무것도 바뀌지 않고 기존 1월이 그대로 남음 ✅
```

그리고 같은 쓰기를 다시 실행해도 결과가 같아서(**멱등**) 재실행에 안전합니다.

# 3. 눈으로 보는 동작 예시

(공식 문서의 예시 구조를 참고해 만든 설명용 예시입니다.)

**교체 전 `orders`**

| order_id | order_dt | note |
|---|---|---|
| 1 | 2026-01-05 | jan-old-a |
| 2 | 2026-01-20 | jan-old-b |
| 3 | 2026-02-05 | feb-keep |

**새로 쓸 데이터 `new_jan`** (모두 1월)

| order_id | order_dt | note |
|---|---|---|
| 10 | 2026-01-08 | jan-new-a |
| 11 | 2026-01-15 | jan-new-b |

```python
(spark.table("new_jan").write
   .format("delta")
   .mode("overwrite")
   .option("replaceWhere", "order_dt >= '2026-01-01' AND order_dt < '2026-02-01'")
   .saveAsTable("main.l0.orders"))
```

**교체 후 `orders`** — 1월 행(1, 2)은 사라지고 새 1월 행(10, 11)이 들어가며, 2월 행(3)은 **그대로**입니다.

| order_id | order_dt | note |
|---|---|---|
| 10 | 2026-01-08 | jan-new-a |
| 11 | 2026-01-15 | jan-new-b |
| 3 | 2026-02-05 | feb-keep |

SQL로는 다음과 같이 씁니다.

```sql
INSERT INTO TABLE main.l0.orders
REPLACE WHERE order_dt >= '2026-01-01' AND order_dt < '2026-02-01'
SELECT * FROM new_jan;
```

# 4. 핵심 동작 규칙

## 4-1. 조건은 임의의 식이고, 파티션 컬럼일 필요가 없다

`REPLACE WHERE`는 **테이블이 그 조건의 컬럼으로 파티션되어 있지 않아도** 됩니다. 조건은 테이블 컬럼에 대한 임의의 식입니다. (예: `colA = 5`, `int_col IN (1, 2, 3)`)

- 과거의 **레거시 replaceWhere**는 **파티션 컬럼 조건만** 허용했고 classic compute에서만 사용할 수 있습니다. 레거시 동작이 필요하면 `spark.databricks.delta.replaceWhere.dataColumns.enabled`를 `false`로 설정합니다.

## 4-2. 쓰는 데이터는 반드시 조건에 맞아야 한다

기본 동작은 **쓰려는 모든 행이 조건에 맞는지 검증**하고, 하나라도 벗어나면 **오류로 중단**합니다. (아무것도 쓰지 않음)

```python
# new_jan 에 2월 행이 섞여 있다면?
# replaceWhere 조건은 1월인데 2월 행이 포함 → 기본적으로 오류로 실패
```

- 이 검증은 **안전장치**입니다. "엉뚱한 범위를 덮어쓰는 사고"를 막아 줍니다.
- classic compute에서는 `spark.databricks.delta.replaceWhere.constraintCheck.enabled`를 `false`로 하면 이 검증을 끌 수 있습니다. 끄면 **조건 범위는 덮어쓰고, 조건 밖의 행은 그냥 삽입**됩니다. **권장하지 않습니다.** (안전장치를 푸는 것이므로)

## 4-3. 소스가 비어 있으면 기존 행이 삭제될 수 있다

공식 문서: "빈 소스 쿼리에서 **REPLACE WHERE는 테이블의 행을 삭제할 수 있다.**" (REPLACE USING / REPLACE ON은 삭제하지 않음)

- 조회 오류나 일시 장애로 원천이 "0건"처럼 보이는 상황에서 그대로 쓰면 **정상 데이터가 지워질 수 있습니다.**
- 대응: 쓰기 전에 소스가 비었는지 확인하고, 정책을 정합니다(건너뛰기 / 정말 비우기).

```python
if df.limit(1).isEmpty():
    print("[WARN] 원천 0건 → 기존 데이터 유지(skip)")
else:
    df.write.format("delta").mode("overwrite") \
      .option("replaceWhere", cond).saveAsTable(target)
```

# 5. 지원 버전과 사용 환경

| 방식 | 지원 |
|---|---|
| **REPLACE WHERE** (Python/Scala) | DBR **9.1 LTS 이상**, 모든 compute 유형 |
| **REPLACE WHERE** (SQL) | DBR **12.2 LTS 이상**, 모든 compute 유형 |

- `replaceWhere`의 조건식은 SQL의 `INSERT`에서 쓸 수 있는 `boolean_expression`이며 일부 제약이 있습니다. (상세는 INSERT SQL 레퍼런스 참고)
- Lakeflow 파이프라인에는 증분 새로고침을 활용하는 **REPLACE WHERE flows**가 있습니다.

# 6. 형제 옵션들과 비교

Databricks의 "선택적 덮어쓰기" 옵션은 여러 개입니다. 문서는 **대부분 REPLACE USING 또는 REPLACE WHERE**를 권장하고, REPLACE ON은 복잡하거나 NULL-safe한 매칭이 필요할 때만 권장합니다.

| 옵션 | 한 줄 설명 | 최소 버전 |
|---|---|---|
| **REPLACE WHERE** | **고정된 조건**에 맞는 행을 교체 (`colA = 5`) | SQL 12.2 LTS+ / Py·Scala 9.1 LTS+ |
| **REPLACE USING** | 제공한 데이터의 **지정 컬럼 값이 같은 행**을 교체(동적) | SQL 16.3+ / Py·Scala 18.2+ |
| **REPLACE ON** | **사용자 정의 조건식**으로 매칭해 교체 (NULL-safe 가능) | SQL 17.1+ / Py·Scala 18.2+ |
| **partitionOverwriteMode** | 레거시 동적 **파티션** 덮어쓰기. 신규 워크로드 비권장 | 11.3 LTS+ |

## REPLACE WHERE vs REPLACE USING — 감각 잡기

- **REPLACE WHERE**: "**내가** 어떤 범위를 갈아엎을지 조건으로 정한다." → 월 단위 재적재처럼 **범위가 미리 정해진** 경우에 자연스럽다.
- **REPLACE USING**: "**들어오는 데이터의 키 값**을 보고, 같은 키의 기존 행을 교체한다." (동적) → 들어오는 데이터가 어떤 키들을 포함하는지에 따라 교체 범위가 달라진다.

REPLACE USING 예시 (공식 문서 구조 참고):

```sql
-- 기존: (1, '2026-01-05'), (2, '2026-01-20'), (3, '2026-02-05')
-- 소스: (1, '2026-01-05') 변경본, (4, '2026-03-01') 신규
INSERT INTO TABLE main.l0.orders
  REPLACE USING (order_id, order_dt)
  SELECT * FROM source_data;
-- → (1, '2026-01-05') 행은 소스 값으로 교체, (4, ...) 는 새로 삽입, (2), (3) 은 그대로
```

- REPLACE USING은 **빈 소스에서 행을 삭제하지 않습니다.**
- DBR 16.3~17.1에서는 REPLACE USING이 **레거시 동작(동적 파티션 덮어쓰기만 허용)** 을 하며, 이때는 USING 절에 **테이블의 파티션 컬럼 전체**를 지정해야 하고 "엉뚱한 파티션의 행 하나가 파티션 전체를 덮어쓸 수 있으므로" 검증이 필요합니다. DBR 17.2 이상에서는 동적 **데이터** 덮어쓰기를 허용합니다.
- REPLACE ON은 `NULL`을 같다고 취급하는 등 REPLACE USING이 지원하지 않는 매칭 로직이 필요할 때 쓰며, `targetAlias` 옵션과 `.as()/.alias()`로 별칭을 지정할 수 있습니다.

# 7. 제약

- Scala/Python에서 `replaceOn`과 `replaceUsing`은 `replaceWhere`, `partitionOverwriteMode`, `overwriteSchema`와 **함께 쓸 수 없습니다.**
- `partitionOverwriteMode`와 `replaceWhere`는 **같은 DataFrameWriter 작업에서 함께 지정할 수 없습니다.**
- 서로 다른 조건의 `replaceWhere`를 **동시에** 실행했을 때의 충돌 여부는 공식 충돌 표에 별도 행이 없습니다. 실제 환경에서 소규모로 테스트하세요. (→ Row-level concurrency 글)

# 8. 실무 체크리스트

- [ ] 조건과 **쓰는 데이터의 범위가 정확히 일치**하는가? (경계는 `>= 시작 AND < 다음시작`으로)
- [ ] 컬럼 타입이 맞는가? (`DATE` 컬럼에 문자열 날짜 비교, `TIMESTAMP`면 시간 포함 범위)
- [ ] 소스가 **0건일 때**의 정책을 정했는가? (REPLACE WHERE는 삭제할 수 있음)
- [ ] 조건 컬럼이 Delta 테이블에 **존재하는 컬럼**인가? (원천 DB의 ROWID 같은 값으로는 조건을 만들 수 없음)
- [ ] 첫 실행(테이블 없음)은 **DDL로 미리 테이블을 만들어 두었는가?**

# 9. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "replaceWhere는 파티션 컬럼에만 쓴다" | 현재 기본 동작은 **임의 조건**, 파티션 불필요 (레거시만 파티션 한정) |
| "조건 밖의 행이 섞여도 알아서 걸러준다" | 기본은 **오류로 중단**. 걸러주지 않음 |
| "빈 데이터면 아무 일도 없다" | REPLACE WHERE는 해당 범위를 **삭제할 수 있다** |
| "`mode("overwrite")`만 쓰면 선택적 덮어쓰기다" | `replaceWhere` 옵션 없이 overwrite하면 **테이블 전체를 덮어씀** |

> 특히 마지막 항목이 위험합니다. `mode("overwrite")`만 쓰면 **전체 교체**입니다. `replaceWhere` 옵션을 빠뜨리지 않았는지 반드시 확인하세요.
{: .notice--warning}

# 10. 한 줄 정리

- `replaceWhere` = **조건에 맞는 행을 원자적으로 교체**. 삭제와 삽입이 **커밋 1번**.
- 실패하면 **기존 데이터가 그대로** 남고, 같은 범위를 다시 실행해도 **중복이 없다**(멱등).
- 쓰는 데이터가 조건 밖이면 **오류**, 소스가 비면 **삭제될 수 있음**.
- 임의 조건이 가능하고 파티션이 필수가 아니다. 대안으로 **REPLACE USING / REPLACE ON**이 있다.
- `mode("overwrite")`에 `replaceWhere`를 **빼먹으면 전체 덮어쓰기**다.

# 관련 용어
Delta 트랜잭션(ACID), 멱등성, 청크(Chunk), Row-level concurrency, 테이블 파티셔닝

## 🔗 참고
- [Selectively overwrite data with Delta Lake](https://docs.databricks.com/aws/en/delta/selective-overwrite)
