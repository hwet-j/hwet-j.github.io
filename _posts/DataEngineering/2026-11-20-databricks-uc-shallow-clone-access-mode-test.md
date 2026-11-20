---
title:  "[DE/Databricks] Unity Catalog Shallow Clone: Standard vs Dedicated 접근 모드별 권한 테스트"
excerpt: "Dedicated 모드에서는 타깃을 변경할 때 소스 테이블의 MODIFY 권한까지 필요하다는 문서 내용을 실제 환경에서 두 모드로 직접 테스트해 확인"

categories:
  - DataEngineering
tags:
  - [Databricks, Unity Catalog, Shallow Clone, Access Mode, Permission, Test]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

직접 테스트한 결과를 일반화해서 정리했습니다. 카탈로그·스키마·테이블·컬럼명과 값은 모두 **예시**입니다.    
공식 문서(AWS 기준)를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[Shallow clone for Unity Catalog tables](https://docs.databricks.com/aws/en/delta/clone-unity-catalog) ·
[Clone a table on Databricks](https://docs.databricks.com/aws/en/delta/clone) ·
[Unity Catalog privileges](https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/privileges) ·
[Compute 구성(접근 모드)](https://docs.databricks.com/aws/en/compute/configure)
{: .notice--warning}

# 1. 상황

운영 테이블을 Unity Catalog의 **Shallow clone**으로 복제해서 테스트용으로 변경(UPDATE 등)하려고 합니다. 그런데 컴퓨트의 **접근 모드**가 **Standard**인지 **Dedicated**인지에 따라 **소스 테이블에 필요한 권한이 달라진다**는 문서 내용이 있어서, 실제 환경에서 두 모드를 모두 테스트했습니다.

# 2. 한눈에 보는 결론

| 항목 | 결과 |
|---|---|
| **Standard** | 소스에 **USE + SELECT**만 있으면 **복제부터 타깃 UPDATE까지 모두 성공**. 소스 `MODIFY` 불필요 (타깃만 변경하므로) |
| **Dedicated** | 소스에 같은 권한(USE + SELECT)만 주면 **최초 복제는 성공**하지만 이후 작업이 실패: `PERMISSION_DENIED ... does not have MODIFY on Table '<SOURCE>'` |
| 문서와의 일치 | **일치**. Dedicated에서 타깃을 변경(insert/update)하려면 **소스에 `MODIFY` 필요** |
| 권장 | **Standard 접근 모드** (소스와 클론의 권한을 독립적으로 관리) |

# 3. 공식 문서 내용

## 3-1. 접근 모드

| 접근 모드 | 설명 |
|---|---|
| **Standard** (구 shared) | **여러 사용자가 사용**할 수 있고 **사용자 간 데이터 격리** |
| **Dedicated** (구 single user) | **단일 사용자 또는 그룹**에 할당해서 사용 |

- 접근 모드는 **누가 컴퓨트를 쓸 수 있고 그 컴퓨트로 어떤 데이터에 접근하는지**를 정하는 보안 기능입니다.
- 기본값 `Auto`는 보통 **Standard**를 고르고, **ML 런타임, GPU 인스턴스, DBR 14.3 미만**을 고르면 **Dedicated**를 씁니다.
- Databricks는 **지원하지 않는 기능이 필요한 경우가 아니면 Standard를 사용**하라고 권장합니다.

## 3-2. Unity Catalog shallow clone의 권한 (접근 모드별)

| | Standard | Dedicated |
|---|---|---|
| 조회 | 타깃에 `SELECT` (+ `USE CATALOG`, `USE SCHEMA`) | 타깃 권한 + **소스 카탈로그·스키마 `USE`, 소스 테이블 `SELECT`** |
| 타깃 변경 (INSERT/UPDATE/DELETE/MERGE/CREATE TABLE/DROP TABLE) | **타깃의 `MODIFY`** | 타깃 권한 + **소스 테이블의 `MODIFY`** |
| 소스 권한 | 독립적으로 관리 가능 | **소스와 타깃 양쪽 리소스에 권한 필요** |

- 복제를 **생성**하려면(관리형): 소스에 `USE CATALOG`, `USE SCHEMA`(그리고 CLONE 문서 기준 `SELECT`), 타깃 스키마에 `USE SCHEMA`와 `CREATE TABLE`, 타깃 카탈로그에 `USE CATALOG`가 필요합니다.
- `SHALLOW CLONE`을 실행한 사용자가 **타깃 테이블의 소유자**가 되어, 소스와 별개로 접근 권한을 통제합니다.
- 문서가 Standard를 권장하는 이유: **Shallow clone(타깃)과 소스의 권한을 서로 독립적으로 바꿀 수 있기 때문**입니다.

## 3-3. `MODIFY` 권한의 의미

- 테이블에서 `MODIFY`는 **INSERT, UPDATE, DELETE** 권한입니다. (`SELECT`, `USE SCHEMA`, `USE CATALOG`도 함께 필요)
- 따라서 **Dedicated에서 소스 `MODIFY`를 주면, 그 주체가 소스(운영) 테이블을 직접 수정할 수도 있게** 됩니다. (4장 해석 참고)

# 4. 테스트

## 4-1. 환경과 권한 설정

| 항목 | Standard | Dedicated |
|---|---|---|
| 타깃(카탈로그·스키마·테이블) | 실행 사용자에게 **모든 권한** | 실행 사용자(또는 지정된 사용자/그룹)에게 **모든 권한** |
| 소스(카탈로그·스키마·테이블) | `USE`, `SELECT`만 (복사 목적) | `USE`, `SELECT`만 (복사 목적) |
| 소스 `MODIFY` | **없음** | **없음** (Dedicated에 지정된 사용자·그룹도 동일) |
| 기록되지 않은 것 | Databricks Runtime 버전, 클라우드, 관리형/외부 테이블 여부, 실행 일자 | 〃 |

두 모드의 **조건은 소스 권한을 똑같이** 맞췄습니다. 달라진 것은 컴퓨트의 접근 모드뿐입니다.

## 4-2. 실행한 코드 (예시 이름)

```python
# ① 타깃 테이블 삭제 (Unity Catalog는 기존 shallow clone을 CREATE OR REPLACE로 덮어쓸 수 없어 먼저 DROP)
spark.sql(f"DROP TABLE IF EXISTS {TARGET_CATALOG}.{TARGET_SCHEMA}.{TARGET_TABLE}")

# ② Shallow clone
spark.sql(f"""CREATE OR REPLACE TABLE {TARGET_CATALOG}.{TARGET_SCHEMA}.{TARGET_TABLE}
              SHALLOW CLONE {SOURCE_CATALOG}.{SOURCE_SCHEMA}.{SOURCE_TABLE}""")

# ③ 타깃 변경
spark.sql(f"UPDATE {TARGET_CATALOG}.{TARGET_SCHEMA}.{TARGET_TABLE} "
          f"SET title = 'test' WHERE id = '000000000001'")

# ④ 확인
spark.sql(f"SELECT * FROM {TARGET_CATALOG}.{TARGET_SCHEMA}.{TARGET_TABLE} WHERE title = 'test'").display()
```

## 4-3. 결과

| 접근 모드 | 소스 MODIFY | 결과 | 오류 |
|---|---|---|---|
| **Standard** | 없음 | **①②③④ 모두 성공** | 없음 |
| **Dedicated** | 없음 | **최초 적재(복제 생성)는 성공, 이후 작업은 실패** | `PERMISSION_DENIED: Principal '<USER OR GROUP>' does not have MODIFY on Table '<SOURCE>'` |

- Standard에서는 **소스에 `MODIFY`가 없어도** 모든 단계가 성공했습니다. 타깃만 변경하기 때문입니다.
- Dedicated에서는 **타깃 변경 작업이 소스의 `MODIFY`를 요구**하며 실패했습니다.

## 4-4. 해석

- 결과는 **공식 문서와 일치**합니다. (Dedicated: 타깃을 변경하는 쿼리는 소스 `MODIFY` 필요, Standard: 타깃 `MODIFY`만 필요)
- 문서는 **왜 Dedicated에서 소스 `MODIFY`가 필요한지 이유를 설명하지 않습니다.** "Dedicated는 소스와 타깃 양쪽의 권한을 모두 평가한다"는 것이 문서 서술이며, 그 이상의 내부 동작은 **추정**입니다.
- **보안 관점**: Dedicated에서 이 오류를 해결하려고 소스에 `MODIFY`를 부여하면 **그 주체가 소스(운영) 테이블을 직접 수정할 수 있게** 됩니다. "운영 데이터를 건드리지 않고 실험한다"는 shallow clone의 목적과 어긋나므로, **Standard 모드를 쓰는 편이 안전**합니다.

## 4-5. 테스트의 한계

| 항목 | 내용 |
|---|---|
| 구문별 결과 | Dedicated에서 **어떤 구문이 실패했는지는 오류 메시지 외에 구문별로 기록하지 않았음.** 문서가 명시한 것은 "타깃에 insert/update 하는 쿼리". `DROP TABLE`, `CREATE OR REPLACE`가 Dedicated에서 어떻게 동작하는지는 **별도 확인 필요** |
| 대조 실험 | Dedicated에 **소스 `MODIFY`를 부여해서 성공하는지**는 확인하지 않음 |
| 환경 | 런타임, 클라우드, 테이블 유형(관리형/외부) 기록 없음. 참고 링크는 Azure 문서였고, 이 글은 **AWS 문서**로 대조(Azure 문서는 접근이 제한되어 확인하지 못함) |
| 반복 | 각 모드 1회 |

# 5. 권한 부여 예시 (Standard 기준)

(일반적인 Unity Catalog 권한 구문의 예시이며 이름은 가상입니다.)

```sql
-- 소스: 복제를 위한 읽기 권한만
GRANT USE CATALOG ON CATALOG src_catalog TO `dev_team`;
GRANT USE SCHEMA  ON SCHEMA  src_catalog.src_schema TO `dev_team`;
GRANT SELECT      ON TABLE   src_catalog.src_schema.src_table TO `dev_team`;

-- 타깃: 클론을 만들고 변경하기 위한 권한
GRANT USE CATALOG  ON CATALOG tgt_catalog TO `dev_team`;
GRANT USE SCHEMA, CREATE TABLE ON SCHEMA tgt_catalog.tgt_schema TO `dev_team`;
-- 클론을 만든 사용자가 타깃 테이블의 소유자가 되므로 이후 타깃 변경이 가능

-- 현재 권한 확인
SHOW GRANTS ON TABLE src_catalog.src_schema.src_table;
```

# 6. 문제 해결 흐름

```
오류: PERMISSION_DENIED ... does not have MODIFY on Table '<SOURCE>'
 └─ 컴퓨트의 접근 모드 확인
     ├─ Dedicated → ① Standard 모드 컴퓨트로 전환 (권장)
     │              ② 불가하면 소스 MODIFY 부여 (운영 테이블 수정 가능해지는 위험 인지)
     └─ Standard  → 타깃의 MODIFY/소유권, USE CATALOG·USE SCHEMA 확인
```

# 7. 이 글의 근거 (무엇을 직접 테스트했고, 무엇이 문서·추정인가)

| # | 주장 | 근거 유형 | 근거 내용 |
|---|---|---|---|
| 1 | Standard에서는 소스 `MODIFY` 없이 복제·UPDATE 모두 성공 | **직접 테스트** | 소스에 `USE`, `SELECT`만 부여하고 ①~④ 성공 |
| 2 | Dedicated에서는 최초 복제 후 작업이 `MODIFY` 부족으로 실패 | **직접 테스트** | `PERMISSION_DENIED ... MODIFY on Table '<SOURCE>'` |
| 3 | 두 모드의 차이는 문서와 일치 | **직접 테스트 + 공식 문서** | UC shallow clone 문서의 접근 모드별 권한 |
| 4 | Databricks는 Standard를 권장 | **공식 문서** | 접근 모드 문서, UC shallow clone 문서 |
| 5 | Dedicated에서 소스 `MODIFY` 부여 시 소스를 수정할 수 있다 | **공식 문서(권한 의미)** | `MODIFY` = INSERT/UPDATE/DELETE. **이 시나리오는 직접 테스트 안 함** |
| 6 | Dedicated에서 소스 권한을 평가하는 내부 이유 | **추정** | 문서에 이유 설명 없음 |
| 7 | Dedicated에서 `DROP`/`CREATE OR REPLACE`도 실패하는지 | **확인 안 됨** | 구문별 기록 없음 |

# 8. 체크리스트

- [ ] 컴퓨트의 **접근 모드**(Standard/Dedicated)를 확인했는가?
- [ ] 소스에 **`USE CATALOG`, `USE SCHEMA`, `SELECT`**가 있는가? 타깃에 **`CREATE TABLE`**과 소유권이 있는가?
- [ ] Dedicated라면 **소스 `MODIFY` 요구**와 **운영 테이블 수정 위험**을 이해했는가?
- [ ] 기존 shallow clone을 갱신할 때 **`DROP` 후 `CREATE`** 하는가? (UC에서는 `CREATE OR REPLACE` 불가)
- [ ] 테스트 기록에 **런타임 버전, 테이블 유형, 구문별 결과**를 남겼는가?

# 9. 한 줄 정리

- UC Shallow clone에서 **Standard는 소스 읽기(`USE`, `SELECT`)만으로 복제와 타깃 변경이 모두 가능**하고, **Dedicated는 타깃을 변경하려면 소스 `MODIFY`가 필요**하다. (직접 테스트로 확인, 문서와 일치)
- 소스 `MODIFY`를 주면 **운영 테이블을 수정할 수 있게 되므로**, Databricks 권고대로 **Standard 모드**를 쓰는 것이 안전하다.
- 구문별 결과, 소스 `MODIFY` 부여 후 대조 실험, 환경 기록은 **다음 테스트에서 보완**한다.

# 관련 용어 글
Unity Catalog 접근 모드(Standard/Dedicated) · Unity Catalog 테이블 권한 · Shallow clone · Deep clone

## 🔗 참고
- [Shallow clone for Unity Catalog tables](https://docs.databricks.com/aws/en/delta/clone-unity-catalog)
- [Clone a table on Databricks](https://docs.databricks.com/aws/en/delta/clone)
- [Unity Catalog privileges and securable objects](https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/privileges)
- [Compute configuration (Access modes)](https://docs.databricks.com/aws/en/compute/configure)
