---
title:  "[DE/용어] Unity Catalog 테이블 권한 (USE CATALOG, USE SCHEMA, SELECT, MODIFY)"
excerpt: "테이블에 접근하려면 왜 카탈로그·스키마 권한까지 필요한지, 그리고 SELECT와 MODIFY의 의미"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, Unity Catalog, Privileges, Permission]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[Unity Catalog privileges and securable objects](https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/privileges)
{: .notice--warning}

# 0. 이 글을 읽기 전에

Unity Catalog에서는 테이블 하나를 읽는 데도 **카탈로그 → 스키마 → 테이블**의 **여러 단계 권한**이 필요합니다. 권한 오류(`PERMISSION_DENIED`)를 이해하려면 이 구조를 알아야 합니다.

# 1. 한 줄 정의

Unity Catalog의 권한은 **카탈로그 > 스키마 > 테이블**의 계층에 부여되고, **하위 객체에 접근하려면 상위 컨테이너의 "사용(USE)" 권한도 함께 필요**합니다.

```
카탈로그 (USE CATALOG)
 └─ 스키마 (USE SCHEMA)
     └─ 테이블 (SELECT, MODIFY ...)
```

# 2. 주요 권한

| 권한 | 대상 | 의미 | 함께 필요한 권한 |
|---|---|---|---|
| **`USE CATALOG`** | 카탈로그 | 카탈로그 안의 객체와 상호작용하기 위한 **사용 권한**. **카탈로그나 그 안의 특정 객체에 대한 접근 자체를 주지는 않음** | — |
| **`USE SCHEMA`** | 스키마 | 스키마 안의 객체에 접근하기 위한 사용 권한 | `USE CATALOG` |
| **`SELECT`** | 테이블·뷰 등 | 객체를 **조회** | 부모 카탈로그 `USE CATALOG` + 부모 스키마 `USE SCHEMA` |
| **`MODIFY`** | 테이블 | 테이블 데이터를 **INSERT, UPDATE, DELETE** | **`SELECT`**, `USE SCHEMA`, `USE CATALOG` |
| **`CREATE TABLE`** | 스키마·카탈로그 | 테이블 생성 | `USE SCHEMA` 등 |
| **`ALL PRIVILEGES`** | 객체 | 그 객체와 **하위 객체**에 대해 **해당 유형에 적용되는 모든 권한**을 의미. 개별 권한을 하나씩 부여하는 것은 아님. **`MANAGE`는 포함되지 않음** | — |

- `USE CATALOG`은 **카탈로그 소유자의 통제 경계**이기도 합니다. 테이블 소유자가 어떤 사용자에게 `SELECT`를 줘도, 그 사용자에게 **부모 카탈로그의 `USE CATALOG`가 없으면** 테이블에 접근할 수 없습니다. 이 권한은 카탈로그 소유자나 `MANAGE`를 가진 사용자만 부여할 수 있습니다.
- 모든 사용자는 기본적으로 `main` 카탈로그에 대한 `USE CATALOG`를 가집니다.

# 3. 권한 상속

- 스키마에 `SELECT`를 부여하면 **그 스키마의 현재와 미래의 모든 테이블·뷰**에 `SELECT`가 부여됩니다.
- 카탈로그에 부여하면 **카탈로그 안 모든 현재·미래의 테이블**에 적용됩니다.
- `MODIFY`도 마찬가지로 스키마·카탈로그 수준에서 부여하면 하위 테이블에 상속됩니다.

> 소스 테이블 하나에만 `MODIFY`를 주려는 의도로 **스키마나 카탈로그에 `MODIFY`를 부여하면, 그 아래 모든 테이블을 수정할 수 있게** 됩니다. 범위를 신중히 정하세요.
{: .notice--warning}

# 4. 권한 확인과 부여 (일반적인 구문 예시)

```sql
SHOW GRANTS ON TABLE main.demo.tbl_src;                        -- 테이블에 부여된 권한 확인
GRANT SELECT ON TABLE main.demo.tbl_src TO `dev_team`;
GRANT USE CATALOG ON CATALOG main TO `dev_team`;
GRANT USE SCHEMA ON SCHEMA main.demo TO `dev_team`;
REVOKE MODIFY ON TABLE main.demo.tbl_src FROM `dev_team`;
```

# 5. 예시: 이런 권한 오류가 나는 이유

| 증상 | 의심할 권한 |
|---|---|
| 테이블이 **보이지 않음/접근 불가** | 상위 **`USE CATALOG` / `USE SCHEMA`** 누락 |
| `SELECT`는 되는데 **UPDATE가 안 됨** | **`MODIFY`** 누락 |
| `MODIFY`를 줬는데 안 됨 | **`SELECT`** 등 함께 필요한 권한 누락 |
| Shallow clone을 Dedicated에서 변경할 때 `MODIFY on Table '<SOURCE>'` 오류 | **소스의 `MODIFY`** (접근 모드 특성) |

# 6. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "테이블 `SELECT`만 있으면 읽을 수 있다" | **`USE CATALOG`, `USE SCHEMA`도 필요** |
| "`MODIFY`만 있으면 수정된다" | **`SELECT`, `USE SCHEMA`, `USE CATALOG`도 필요** |
| "`ALL PRIVILEGES`는 모든 걸 준다" | **`MANAGE`는 포함되지 않음** |
| "`USE CATALOG`는 카탈로그 접근을 준다" | **접근 자체는 주지 않는** 사용 권한 |

# 7. 한 줄 정리

- 읽기는 **`USE CATALOG` + `USE SCHEMA` + `SELECT`**, 쓰기는 거기에 **`MODIFY`(SELECT 포함 필요)** 가 필요하다.
- 권한은 **상위 → 하위로 상속**되므로 부여 범위를 신중히 정한다. `MODIFY`는 **INSERT/UPDATE/DELETE**를 의미한다.

# 관련 용어
컴퓨트 접근 모드(Standard/Dedicated), Shallow clone

## 🔗 참고
- [Unity Catalog privileges and securable objects](https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/privileges)
