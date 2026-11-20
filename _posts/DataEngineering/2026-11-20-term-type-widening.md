---
title:  "[DE/용어] Type widening (컬럼 타입 넓히기)"
excerpt: "데이터 파일을 다시 쓰지 않고 컬럼 타입을 INT→BIGINT처럼 넓히는 기능"

categories:
  - DataEngineering
tags:
  - [Glossary, Delta Lake, Schema, Type Widening]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[Type widening](https://docs.databricks.com/aws/en/tables/features/type-widening)    
[Delta Lake feature compatibility and protocols](https://docs.databricks.com/aws/en/delta/feature-compatibility)
{: .notice--warning}

# 0. 이 글을 읽기 전에

"기존 테이블의 컬럼 타입을 INT에서 BIGINT로 바꾸고 싶은데 데이터를 전부 다시 써야 하나?"에 대한 답입니다. **스키마 강제**, **스키마 진화** 글을 보시면 좋습니다.

# 1. 한 줄 정의

**Type widening**은 Delta 테이블에서 컬럼 타입을 **더 넓은 타입으로 바꾸는 것을, 데이터 파일을 다시 쓰지 않고** 할 수 있게 해 주는 기능입니다. **Databricks Runtime 15.4 LTS 이상**의 Delta 테이블에서 사용할 수 있습니다.

# 2. 왜 필요한가

- 처음에 `row_cnt INT`로 만들었는데 데이터가 커져서 `BIGINT`가 필요해졌습니다.
- 기존 방식은 **테이블을 통째로 다시 쓰기**(`overwriteSchema`)였습니다. 큰 테이블에서는 비용이 큽니다.
- Type widening은 **메타데이터만 바꿔서** 같은 일을 합니다. (데이터 파일은 그대로)

# 3. 어떤 변경이 가능한가 (공식 문서)

| 원래 타입 | 넓힐 수 있는 타입 |
|---|---|
| BYTE | SHORT, INT, BIGINT |
| SHORT | INT, BIGINT |
| **INT** | **BIGINT** |
| FLOAT | DOUBLE |
| DECIMAL | 정밀도·스케일이 더 큰 DECIMAL |
| DATE | TIMESTAMP_NTZ |
| VOID | 모든 타입 |

- **DBR 15.4.3 이상**에서는 BYTE/SHORT/INT/BIGINT → **DECIMAL**, BYTE/SHORT/INT → **DOUBLE**로의 변경은 **불가**합니다.
- 최상위 컬럼뿐 아니라 **struct, map, array 안의 필드**도 지원합니다.
- DECIMAL: 전체 정밀도는 **시작 정밀도 이상**이어야 하고, 스케일을 늘리면 정밀도도 그만큼 늘려야 합니다. 예: `decimal(10,1)`에 소수 두 자리를 더하려면 최소 `decimal(12,3)`.
- VOID → 임의 타입은 type widening을 켜지 않아도 되고, DBR 18.2 이상에서 사용 가능합니다.

# 4. 사용 방법

## 4-1. 켜기

```sql
-- 기존 테이블
ALTER TABLE main.l0.job_hist SET TBLPROPERTIES ('delta.enableTypeWidening' = 'true');

-- 생성 시
CREATE TABLE t (c1 INT) TBLPROPERTIES ('delta.enableTypeWidening' = 'true');
```

## 4-2. 수동으로 타입 변경

```sql
ALTER TABLE main.l0.job_hist ALTER COLUMN row_cnt TYPE BIGINT;
```

- 데이터 파일을 **다시 쓰지 않고** 스키마만 업데이트합니다.

## 4-3. 쓰기 중 자동으로 넓히기 (스키마 진화와 함께)

```python
# 대상 테이블: id INT (type widening 켜짐), 소스: id BIGINT
spark.table("source_table").write.mode("append").option("mergeSchema", "true").saveAsTable("target_table")
```

```sql
INSERT WITH SCHEMA EVOLUTION INTO target_table SELECT * FROM source_table;
```

자동으로 넓히려면 **모두** 만족해야 합니다.

1. 쓰기가 **스키마 진화가 켜진 상태**로 실행
2. **대상 테이블에 type widening이 켜져** 있음
3. **소스 컬럼 타입이 대상보다 넓음**
4. type widening이 그 변경을 지원

조건에 맞지 않으면 **일반 스키마 강제 규칙**을 따릅니다. 또한 **type widening이 꺼져 있으면 스키마 진화는 대상 타입에 맞게 downcast를 시도**하므로, 타입을 자동으로 넓히고 싶지 않으면 스키마 진화를 켠 워크로드 전에 type widening을 끄라고 안내합니다.

# 5. 방향 정리 (가장 많이 헷갈리는 부분)

```
[type widening 이 해 주는 일]   테이블 INT(좁음)  ← 데이터 BIGINT(넓음)   → 테이블 타입을 BIGINT 로 넓힘
[해 주지 않는 일]              테이블 BIGINT(넓음) ← 데이터 INT(좁음)     → 이미 테이블이 넓으므로 대상 아님
```

- Type widening은 **테이블 컬럼의 타입을 넓히는** 기능입니다. 이미 BIGINT인 테이블에 INT DataFrame을 쓸 때 생기는 불일치를 풀어 주는 기능이 **아닙니다.** (→ INT DataFrame을 BIGINT 테이블에 쓸 때 사례 글)

# 6. 중요한 제약: 프로토콜과 호환성

> 공식 문서: type widening을 켜면 **reader/writer 프로토콜이 업그레이드**됩니다. 외부 Delta Lake 클라이언트와의 **호환성에 영향**이 있을 수 있고, type widening이 켜진 테이블은 **Databricks Runtime 15.4 LTS 이상에서만 읽을 수 있습니다.**
{: .notice--warning}

- 테이블을 읽는 **BI 도구, 오래된 클러스터, 외부 시스템**이 있다면 켜기 전에 확인해야 합니다. (→ Delta 프로토콜 글)
- **DBR 15.4 LTS에서 켠 테이블**은 `typeWidening-preview` 기능을 쓰므로, 제거할 때 `DROP FEATURE`의 대상이 다릅니다.

# 7. 끄기와 제거

```sql
-- 앞으로의 타입 변경 방지 (기능이 제거되거나 이전 변경이 취소되지는 않음)
ALTER TABLE main.l0.job_hist SET TBLPROPERTIES ('delta.enableTypeWidening' = 'false');

-- 기능 자체를 프로토콜에서 제거
ALTER TABLE main.l0.job_hist DROP FEATURE 'typeWidening' [TRUNCATE HISTORY];
```

- **끄는 것(false)** 은 "이후 변경을 막는 것"일 뿐 **프로토콜을 내리지 않습니다.** 완전히 제거하려면 `DROP FEATURE`가 필요합니다.
- `DROP FEATURE` 시 Databricks는 **현재 테이블 스키마에 맞지 않는 모든 데이터 파일을 다시 씁니다.**

# 8. 스트리밍·Auto Loader

- Structured Streaming에서의 type widening은 **DBR 16.4 LTS 이상**에서 지원됩니다. 소스 Delta 테이블에서 스트리밍할 때 **대상 테이블에 스키마 진화(`mergeSchema`)와 type widening**이 모두 켜져 있어야 합니다.
- **Auto Loader**의 type widening 지원은 **Public Preview**입니다.

# 9. 선택 가이드: 컬럼 타입을 바꾸고 싶다면

| 상황 | 방법 |
|---|---|
| 큰 테이블, INT→BIGINT처럼 **넓히기**, DBR 15.4 LTS+ 환경 | **Type widening** (`ALTER COLUMN TYPE`) |
| **프로토콜 상승 불가**(외부 시스템 호환 문제) | 테이블 재생성 또는 `overwriteSchema` (전체 재작성) |
| 좁히거나 임의 변환 | `overwriteSchema` 로 재작성 |
| **들어오는 데이터에 맞춰 자동으로** 넓히고 싶음 | 스키마 진화 + type widening (조건 4가지) |
| 단지 **쓰기 오류를 피하고** 싶음 (테이블은 그대로) | DataFrame을 **대상 타입으로 cast** |

# 10. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "type widening을 켜면 INT/BIGINT 불일치 쓰기 오류가 사라진다" | 방향이 다름. **테이블이 더 넓은 경우는 해당 없음** |
| "켜는 건 부작용이 없다" | **프로토콜이 올라가** DBR 15.4 LTS 미만에서 읽지 못함 |
| "false로 끄면 원래대로 돌아간다" | 이전 변경·프로토콜은 그대로. 제거는 `DROP FEATURE` |
| "모든 숫자 타입 변환이 가능" | INT→DECIMAL, INT→DOUBLE 등은 DBR 15.4.3+에서 불가 |

# 11. 한 줄 정리

- Type widening = **데이터 파일을 다시 쓰지 않고 컬럼 타입을 더 넓은 타입으로 바꾸는** 기능 (DBR 15.4 LTS+).
- `delta.enableTypeWidening`을 켜고 `ALTER COLUMN ... TYPE`으로 변경하거나, **스키마 진화와 함께** 들어오는 넓은 데이터에 맞춰 자동 확장한다.
- **테이블이 좁고 데이터가 넓은** 방향에만 해당한다.
- 켜면 **프로토콜이 올라가** 호환성에 영향. 제거는 `DROP FEATURE`.

# 관련 용어
스키마 강제(Schema enforcement), 스키마 진화(Schema evolution), Delta 프로토콜과 Table feature

## 🔗 참고
- [Type widening](https://docs.databricks.com/aws/en/tables/features/type-widening)
- [Delta Lake feature compatibility and protocols](https://docs.databricks.com/aws/en/delta/feature-compatibility)
