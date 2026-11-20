---
title:  "[DE/용어] Deep clone"
excerpt: "소스 테이블의 데이터와 메타데이터를 모두 복사해 소스와 독립된 복사본을 만드는 CLONE"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, Delta Lake, Clone, Deep Clone]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[Clone a table on Databricks](https://docs.databricks.com/aws/en/delta/clone)    
[CREATE TABLE CLONE](https://docs.databricks.com/aws/en/sql/language-manual/delta-clone)
{: .notice--warning}

# 0. 이 글을 읽기 전에

"운영 테이블을 통째로 백업해 두고 싶다"는 요구에 쓰는 기능입니다. 얕은 복제(**Shallow clone**)와 비교하면 이해가 쉽습니다.

# 1. 한 줄 정의

**Deep clone**은 소스 테이블의 **데이터와 메타데이터를 모두 복사**해서, **소스에 의존하지 않는 독립된 테이블**을 만드는 `CLONE`입니다. 특정 **버전** 시점의 복사본을 만들 수 있습니다. `CLONE`의 **기본값**이 Deep입니다.

# 2. 사용법

```sql
CREATE TABLE main.demo.tbl_copy CLONE main.demo.tbl_src;                   -- 기본 = deep
CREATE TABLE main.demo.tbl_copy DEEP CLONE main.demo.tbl_src;
CREATE OR REPLACE TABLE main.demo.tbl_copy DEEP CLONE main.demo.tbl_src;   -- 기존 타깃 대체
CREATE TABLE IF NOT EXISTS main.demo.tbl_copy CLONE main.demo.tbl_src;     -- 있으면 건너뜀
CREATE TABLE main.demo.model_dataset CLONE main.demo.entire_dataset VERSION AS OF 15;
```

```python
from delta.tables import DeltaTable
DeltaTable.forName(spark, "main.demo.tbl_src").clone(target="main.demo.tbl_copy", isShallow=False, replace=False)
```

# 3. 무엇이 복사되나

| 복사됨 | 복사되지 않음 |
|---|---|
| **데이터 파일** | 테이블 설명 |
| 스키마, 파티셔닝, invariants, nullability, `TBLPROPERTIES` | 사용자 정의 커밋 메타데이터 |
| **스트림 메타데이터, `COPY INTO` 메타데이터** (Deep에서만) | **Delta 테이블 이력** |
| | Unity Catalog 속성(태그 등) |

- 스트림 메타데이터가 복사되므로, **소스에 쓰던 스트림을 중단하고 클론 타깃에서 이어서 실행**할 수 있습니다.
- 클론은 **독립된 이력**을 가지므로, 소스와 같은 입력으로 time travel이 되지 않습니다.
- **스트리밍 테이블과 materialized view는 CLONE의 소스·타깃이 될 수 없습니다.**

# 4. 특징과 비용

| 항목 | 내용 |
|---|---|
| 소스 의존 | **없음** (소스에서 VACUUM하거나 삭제해도 영향 없음) |
| 생성 비용 | 데이터를 **복사하므로 비쌈** (시간, 저장 공간) |
| 저장 공간 | 소스와 비슷한 크기 필요 |
| 권한(클라우드) | 클론 디렉터리 읽기/쓰기 |
| 필요한 테이블 권한 | 소스 **SELECT**, 새 테이블이면 스키마 **CREATE**, 대체하면 **MODIFY** |

## 직접 측정에서 본 경향 (소규모)

크기 50~450MB, 16개 테이블을 측정한 결과 **Deep 시간은 크기보다 파일 개수에 비례**했습니다. (R² 0.96 vs 0.03, 1회 측정, 이상치 2개 제외, 소규모 한정) 자세한 내용은 *Deep Clone vs Shallow Clone* 글을 참고하세요.

# 5. 증분 동기화 (기존 타깃에 다시 CLONE)

기존 Delta 타깃에 **CLONE을 다시 실행**하면 **마지막 클론 이후의 새 메타데이터와 데이터만** 담은 **증분 커밋**이 만들어집니다. 문서 예시는 **한 달에 한 번** 아카이브를 동기화하는 것입니다.

```sql
CREATE OR REPLACE TABLE archive_table CLONE my_prod_table;   -- 주기적으로 실행해 아카이브를 최신으로 유지
```

- 별도 파이프라인 로직이 아니라 **이 명령을 스케줄로 실행**하는 방식입니다.

# 6. 보관 정책 조정

```sql
CREATE OR REPLACE TABLE archive_table CLONE prod.my_table
TBLPROPERTIES (
  delta.logRetentionDuration = '3650 days',
  delta.deletedFileRetentionDuration = '3650 days'
);
```

- 아카이브에서 **time travel이 필요**하면 데이터·로그 보존 기간을 **원본과 독립적으로** 늘릴 수 있습니다.

# 7. 언제 쓰나

| 용도 | 설명 |
|---|---|
| **데이터 보관(archiving)** | 특정 시점 상태를 보존 |
| **재해 복구** | 증분 동기화로 최신 상태 유지 |
| **ML 모델 재현** | 학습에 쓴 버전을 `VERSION AS OF`로 보관 |
| 소스에 **의존하지 않아야** 할 때 | 소스 삭제·VACUUM 영향을 받지 않음 |

# 8. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "CTAS와 같다" | CLONE은 **메타데이터(파티셔닝, 포맷, invariants 등)도 복사**. CTAS는 직접 지정 |
| "이력까지 복제된다" | **이력은 복제되지 않음** |
| "증분 동기화에 별도 파이프라인이 필요하다" | **CLONE 재실행**으로 증분 커밋 |
| "Deep가 항상 크기에 비례해 느리다" | 소규모 측정에서는 **파일 개수의 영향이 더 컸음** (대규모는 미측정) |

# 9. 한 줄 정리

- Deep clone = **데이터까지 복사한 독립 복사본**. 보관·DR·ML 재현에 쓰고, 증분 동기화는 **CLONE 재실행**.
- **이력·태그·설명은 복제되지 않고**, 스트리밍 테이블·MV는 대상이 아니다.

# 관련 용어
Shallow clone, Delta 트랜잭션(ACID)

## 🔗 참고
- [Clone a table on Databricks](https://docs.databricks.com/aws/en/delta/clone)
- [CREATE TABLE CLONE](https://docs.databricks.com/aws/en/sql/language-manual/delta-clone)
