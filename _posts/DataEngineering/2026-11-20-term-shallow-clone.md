---
title:  "[DE/용어] Shallow clone"
excerpt: "메타데이터만 복사하고 소스의 데이터 파일을 참조하는 가볍고 빠른 CLONE과 그 제약"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, Delta Lake, Clone, Shallow Clone]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[Clone a table on Databricks](https://docs.databricks.com/aws/en/delta/clone)    
[Shallow clone for Unity Catalog tables](https://docs.databricks.com/aws/en/delta/clone-unity-catalog)
{: .notice--warning}

# 0. 이 글을 읽기 전에

"운영 테이블에 영향 없이 잠깐 실험해 보고 싶다"는 요구에 쓰는 가벼운 복제입니다. **Deep clone**과 비교하면 이해가 쉽습니다.

# 1. 한 줄 정의

**Shallow clone**은 소스 테이블의 **메타데이터만 복사**하고 **데이터 파일은 복사하지 않은 채 소스의 파일을 참조**하는 `CLONE`입니다. 만드는 비용(컴퓨트·저장 공간)이 적습니다.

```
[Deep]    새 테이블 ── 데이터 파일(복사본) 보유
[Shallow] 새 테이블 ── 소스의 데이터 파일을 "참조" (복사 안 함)
```

# 2. 사용법

```sql
CREATE TABLE main.demo.tbl_test SHALLOW CLONE main.demo.tbl_prod;
CREATE TABLE main.demo.tbl_test SHALLOW CLONE main.demo.tbl_prod VERSION AS OF 15;
CREATE TABLE main.demo.tbl_test SHALLOW CLONE main.demo.tbl_prod TIMESTAMP AS OF '2025-12-01';
```

- 클론에 `UPDATE` 등을 실행해도 **소스에는 영향이 없습니다.** 문서의 "프로덕션 테이블 단기 실험" 예시는 클론에서 검증한 뒤 필요한 변경만 소스에 `MERGE`로 반영합니다.
- 변경하지 않은 데이터는 소스 파일을 **그대로 참조**합니다.

# 3. 핵심 특징과 위험

| 항목 | 내용 |
|---|---|
| 복사 | **메타데이터만**. CLONE 결과의 `num_copied_files`는 **0** |
| 생성 비용 | 컴퓨트·저장 공간이 **적음** (직접 측정에서도 Deep보다 대체로 빠름) |
| 소스 의존 | **있음** |
| 클라우드 권한 | 클론 디렉터리 + **소스의 데이터 파일**을 읽을 권한 |
| 복제 안 됨 | 설명, 사용자 정의 커밋 메타데이터, **Delta 이력**, UC 태그 |

## ⚠️ 소스에서 VACUUM하면?

| 환경 | 동작 (공식 문서) |
|---|---|
| **Hive metastore / 테이블로 등록되지 않은 파일** | 소스에서 `VACUUM`하면 **참조하던 데이터 파일을 읽을 수 없어 `FileNotFoundException`** 발생. **`clone`을 `replace`로 다시 실행해서 복구**. 자주 생기면 **Deep clone**을 고려 |
| **Unity Catalog** | UC가 **클론과 소스 파일의 관계를 추적**하므로 **소스에서 VACUUM해도 클론이 깨지지 않음** (클론이 필요한 파일은 유지) |

# 4. Unity Catalog에서의 Shallow clone

| 항목 | 내용 |
|---|---|
| 상태 | **Public Preview** |
| 런타임 | 관리형 **DBR 13.3 LTS+**, 외부 **DBR 14.3 LTS+** |
| 대상 | **관리형→관리형**, **외부→외부**. **Delta 테이블만** |
| 권한 독립 | 클론 소유자가 소스와 **별개로 접근 권한을 통제** |
| 제약 | 기존 shallow clone을 `CREATE OR REPLACE`로 **덮어쓸 수 없음**, **중첩 shallow clone 불가**, **OpenSharing 공유 불가** |

## 접근 모드별 권한

| 모드 | 필요한 권한 |
|---|---|
| **Standard**(구 shared) | 조회: 클론의 `SELECT`(+ USE CATALOG/SCHEMA). 변경(INSERT/UPDATE/DELETE/MERGE 등): **클론(타깃)의 MODIFY**. 소스 권한 불필요 |
| **Dedicated**(구 single user) | 소스와 타깃 **양쪽** 권한 필요. 타깃을 변경하려면 **소스의 MODIFY**도 필요 |

- Databricks는 소스와 클론의 권한을 **독립적으로 바꿀 수 있는 Standard 모드**를 권장합니다.

## 베이스(소스) 테이블을 삭제하면?

- **관리형**: 베이스를 삭제해도 소스의 데이터 파일이 지워지기 전까지 클론은 읽힙니다. 복구 기간(기본 7일, `UNDROP`) 이후 파일이 삭제되면 **클론을 쓸 수 없게 됩니다.**
- 일부 워크스페이스는 shallow clone이 있는 베이스 삭제를 **막는 보호 기능**이 있고, `DROP TABLE ... FORCE`로 지우면 **참조하는 클론이 깨집니다.**
- **외부**: `DROP TABLE`은 데이터 파일을 지우지 않으므로 클론이 영향받지 않습니다.

# 5. 직접 측정에서 본 경향 (소규모)

16개 테이블(50~450MB)에서 Shallow는 **파일 개수가 늘어도 시간이 거의 평평**했습니다(1.76~2.29초대, 이상치 제외). 다만 **1회 측정, 소규모, 이상치 2개**라는 한계가 있습니다. 자세한 내용은 *Deep Clone vs Shallow Clone* 글을 참고하세요.

# 6. 언제 쓰나

| 용도 | 설명 |
|---|---|
| **프로덕션 테이블 단기 실험** | 운영 데이터에 영향 없이 변경·검증 |
| 특정 버전/시점의 가벼운 참조본 | `VERSION AS OF`, `TIMESTAMP AS OF` |
| **쓰지 말아야 할 때** | 소스와 **독립**되어야 하는 장기 보관·DR (Deep 사용), 소스 파일에 **읽기 권한이 없는** 사용자에게 제공 |

# 7. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "소스에서 VACUUM하면 항상 깨진다" | **비-UC에서만.** UC는 깨지지 않음 |
| "Dedicated에서도 소스 권한은 필요 없다" | **소스 MODIFY**가 필요(타깃 변경 시) |
| "shallow clone을 다시 shallow clone할 수 있다" | UC에서는 **불가** |
| "저장 공간을 전혀 안 쓴다" | 메타데이터와 이후 **클론에서 쓴 변경분**은 저장됨 |

# 8. 한 줄 정리

- Shallow clone = **메타데이터만 복사하고 소스 파일을 참조**하는 가벼운 복제. **프로덕션 테이블 단기 실험**에 적합.
- **비-UC에서는 소스 VACUUM이 클론을 깨뜨리고**, **UC에서는 깨지지 않는다.** UC에서는 관리형↔관리형 등 제약과 접근 모드별 권한이 있다.

# 관련 용어
Deep clone, Delta 트랜잭션(ACID)

## 🔗 참고
- [Clone a table on Databricks](https://docs.databricks.com/aws/en/delta/clone)
- [Shallow clone for Unity Catalog tables](https://docs.databricks.com/aws/en/delta/clone-unity-catalog)
- [CREATE TABLE CLONE](https://docs.databricks.com/aws/en/sql/language-manual/delta-clone)
