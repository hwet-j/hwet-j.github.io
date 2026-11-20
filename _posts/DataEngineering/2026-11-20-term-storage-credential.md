---
title:  "[DE/용어] Storage Credential"
excerpt: "클라우드 스토리지에 접근하는 인증·인가 수단(AWS IAM 역할, Azure Managed Identity)을 담은 Unity Catalog 객체"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, Unity Catalog, Storage Credential, IAM, Managed Identity]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[Create a storage credential and external location for S3](https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/s3/s3-external-location-manual)    
[Azure: Use Azure managed identities in Unity Catalog](https://learn.microsoft.com/en-us/azure/databricks/connect/unity-catalog/cloud-storage/azure-managed-identities)
{: .notice--warning}

# 0. 이 글을 읽기 전에

**External Location**이 "경로 + 자격"의 묶음이라면, **Storage Credential**은 그중 "자격" 부분입니다.

# 1. 한 줄 정의

**Storage Credential**은 클라우드 테넌트의 데이터에 접근하기 위한 **인증·인가 수단**을 나타내는 Unity Catalog 객체입니다. 클라우드별로 다음에 대응합니다.

| 클라우드 | Storage Credential이 가리키는 것 |
|---|---|
| **AWS (S3)** | **IAM 역할** |
| **Azure (ADLS Gen2)** | **Access Connector for Azure Databricks**의 **Managed Identity** |
| Cloudflare R2 | R2 API 토큰 |

- 이 credential을 써서 External Location을 만들 수 있는 사용자는 **UC 권한**으로 정합니다. 생성·사용 권한은 **External Location을 만들어야 하는 사용자에게만** 줍니다.

# 2. 만드는 방법

## AWS (IAM 역할)

1. AWS에서 **IAM 역할**을 만들고 **임시 신뢰 정책(자리표시자 external ID `0000`)** 을 붙임
2. Databricks에서 **AWS IAM Role** 유형으로 credential 생성 → **External ID 복사**
3. **신뢰 정책**에 복사한 external ID와 **자기 자신(역할) ARN**을 추가 → **self-assuming**
4. **Validate Configuration**으로 검증

## Azure (Managed Identity)

1. **Access Connector for Azure Databricks** 생성
2. Storage Account에서 Access Connector의 Managed Identity에 **`Storage Blob Data Contributor`** 역할 부여
3. Databricks에서 **Azure Managed Identity** 유형으로 credential 생성 (**Access connector ID** 입력)
4. 계정 관리자가 아니면 Access Connector에 대한 **Contributor 역할**이 필요

# 3. 필요한 권한

| 작업 | 필요 권한 |
|---|---|
| Storage Credential 생성 | 메타스토어의 **`CREATE STORAGE CREDENTIAL`** (메타스토어 관리자는 기본 보유) |
| External Location 생성 | 메타스토어와 해당 credential의 **`CREATE EXTERNAL LOCATION`** |
| AWS IAM 역할 생성 | **`iam:CreateRole`** |

# 4. 옵션과 운영

- **읽기 전용(Limit to read-only use)**: 이 credential을 쓰는 External Location을 읽기 전용으로 제한. IAM 역할이 쓰기를 허용해도 Databricks가 **무시**합니다. 나중에 변경할 수 있습니다.
- **워크스페이스 바인딩**: 기본값은 메타스토어에 연결된 모든 워크스페이스에서 사용 가능. 특정 워크스페이스에만 쓰도록 제한할 수 있습니다.
- **하나의 credential, 여러 External Location**: 같은 credential이 허용하는 범위 안에서 경로별로 External Location을 나눌 수 있습니다.

# 5. AWS 주의: self-assuming

- AWS가 2023-06-30에 IAM 역할이 **명시적으로 자신을 신뢰**하도록 요구했고, Databricks는 **2025-01-20부터 self-assuming이 아닌 IAM 역할 기반 credential 사용을 차단**했습니다.
- `Validate Configuration`에서 **Self Assume Role** 검사가 실패하면 신뢰 정책에 자기 자신의 ARN이 있는지 확인합니다.

# 6. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "credential = 접근 키를 저장한 것" | **IAM 역할·Managed Identity를 가리키는 객체** (사용자가 키를 몰라도 됨) |
| "credential만 만들면 접근된다" | **External Location을 만들고 GRANT**해야 사용자가 접근 |
| "IAM 역할만 만들면 된다 (AWS)" | **external ID 신뢰 정책과 self-assuming**까지 설정해야 검증 통과 |

# 7. 한 줄 정리

- Storage Credential = **AWS IAM 역할 / Azure Managed Identity**를 가리키는 **스토리지 접근 자격** 객체. External Location이 이를 참조한다.
- AWS는 **external ID + self-assuming**, Azure는 **Access Connector + Storage Blob Data Contributor**가 핵심이다.

# 관련 용어
External Location, DBFS 마운트

## 🔗 참고
- [Create a storage credential and external location for S3](https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/s3/s3-external-location-manual)
- [Azure: Use Azure managed identities in Unity Catalog](https://learn.microsoft.com/en-us/azure/databricks/connect/unity-catalog/cloud-storage/azure-managed-identities)
