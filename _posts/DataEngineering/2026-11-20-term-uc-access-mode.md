---
title:  "[DE/용어] 컴퓨트 접근 모드: Standard vs Dedicated"
excerpt: "컴퓨트를 누가 쓰고 어떤 데이터에 접근하는지를 정하는 보안 설정과 두 모드의 차이"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, Unity Catalog, Access Mode, Compute]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[Compute configuration (Access modes)](https://docs.databricks.com/aws/en/compute/configure)    
[Shallow clone for Unity Catalog tables](https://docs.databricks.com/aws/en/delta/clone-unity-catalog)
{: .notice--warning}

# 0. 이 글을 읽기 전에

Databricks에서 클러스터(컴퓨트)를 만들 때 **Access mode**(접근 모드)를 고르는 항목이 있습니다. 단순한 옵션처럼 보이지만 **권한 동작이 달라질 수 있어서** 장애 원인이 되기도 합니다. (예: Shallow clone에서 소스 `MODIFY` 권한 요구)

# 1. 한 줄 정의

**접근 모드(Access mode)** 는 **누가 그 컴퓨트를 쓸 수 있는지, 그리고 그 컴퓨트로 어떤 데이터에 접근할 수 있는지를 정하는 보안 기능**입니다. 모든 컴퓨트는 접근 모드를 가집니다.

# 2. 두 가지 모드

| 모드 | 이전 이름 | 설명 | 지원 언어 |
|---|---|---|---|
| **Standard** | shared access mode | **여러 사용자**가 쓸 수 있고 **사용자 간 데이터 격리** | Python, SQL, Scala |
| **Dedicated** | single user access mode | **단일 사용자 또는 그룹**에 할당해서 사용 | Python, SQL, Scala, R |

# 3. 기본 선택 규칙 (공식 문서)

- 접근 모드 선택은 기본적으로 **Auto**이고, 선택한 런타임에 따라 자동으로 정해집니다.
- Auto는 보통 **Standard**를 쓰되, **ML 런타임, GPU 인스턴스 유형, DBR 14.3 미만**을 선택하면 **Dedicated**를 씁니다.
- Databricks는 **필요한 기능이 지원되지 않는 경우가 아니면 Standard 사용**을 권장합니다.

# 4. 왜 중요한가: 권한 동작이 다르다

대표적인 예가 **Unity Catalog Shallow clone**입니다.

| | Standard | Dedicated |
|---|---|---|
| 조회 | 타깃의 `SELECT` 등 | 타깃 권한 + **소스의 `USE`, `SELECT`** |
| 타깃 변경 | **타깃의 `MODIFY`** | 타깃 권한 + **소스의 `MODIFY`** |
| 소스·타깃 권한 | **독립적으로 관리** | **양쪽 리소스 권한이 모두 필요** |

- 직접 테스트에서도 소스 `USE`, `SELECT`만 있을 때 **Standard는 모두 성공**, **Dedicated는 타깃 변경 시 `PERMISSION_DENIED ... MODIFY on Table '<SOURCE>'`** 로 실패했습니다. (→ UC Shallow clone 접근 모드 테스트 글)
- 그래서 문서는 Unity Catalog clone 작업에 **Standard 모드**를 권장합니다.

# 5. 어떤 모드를 고를까

| 상황 | 선택 |
|---|---|
| 일반적인 분석·ETL, 여러 사용자가 공유 | **Standard** (권장) |
| **R**이 필요하거나 Standard에서 **지원되지 않는 기능**이 필요 | **Dedicated** |
| **ML 런타임, GPU** | Dedicated (Auto가 선택) |
| 특정 사용자/그룹 전용 컴퓨트 | Dedicated |
| Unity Catalog 테이블을 **독립 권한**으로 복제 | **Standard** |

# 6. 운영 시 참고

- **Dedicated**에서 컴퓨트 로그를 **볼륨**으로 전달하는 기능은 **사용자에게 할당된 경우만** 지원되고 **그룹에 할당된 Dedicated는 지원되지 않습니다.**
- 모드별 기능 지원은 각 모드의 **요구 사항과 제한** 문서에서 확인합니다.

# 7. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "접근 모드는 성능 옵션이다" | **보안 기능**이며 권한·격리 동작이 달라짐 |
| "Dedicated가 더 안전하니 항상 좋다" | 문서는 **Standard를 권장**. Dedicated는 필요한 기능이 있을 때 |
| "모드가 달라도 같은 권한이면 같은 결과" | **Shallow clone처럼 모드에 따라 추가 권한을 요구**하는 경우가 있음 |

# 8. 한 줄 정리

- 접근 모드 = **컴퓨트 사용자와 데이터 접근 범위를 정하는 보안 설정**. **Standard**(여러 사용자, 격리, 권장)와 **Dedicated**(단일 사용자/그룹)가 있다.
- 같은 권한이어도 **모드에 따라 결과가 달라질 수 있다.** (예: Shallow clone의 소스 `MODIFY`)

# 관련 용어
Unity Catalog 테이블 권한, Shallow clone

## 🔗 참고
- [Compute configuration (Access modes)](https://docs.databricks.com/aws/en/compute/configure)
- [Shallow clone for Unity Catalog tables](https://docs.databricks.com/aws/en/delta/clone-unity-catalog)
