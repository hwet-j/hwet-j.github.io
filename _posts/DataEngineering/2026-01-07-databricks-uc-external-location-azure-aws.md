---
title:  "[DE/Databricks] Unity Catalog External Location 등록하기: Azure와 AWS 설정 방법"
excerpt: "Storage Credential과 External Location의 관계, Azure(Access Connector)와 AWS(IAM Role) 각각의 등록 절차, 권한 부여와 문제 해결을 공식 문서 기준으로 정리"

categories:
  - DataEngineering
tags:
  - [Databricks, Unity Catalog, External Location, Storage Credential, Azure, AWS]

toc: true
toc_sticky: true
published: true

date: 2026-01-07
last_modified_at: 2026-11-20
---

업무 중 정리한 설정 절차를 일반화해서 정리했습니다. 계정·버킷·리소스 이름은 모두 **예시**(`<...>` 자리표시자)입니다.    
**AWS**는 공식 문서(Databricks on AWS)를 직접 읽고 정리했고, **Azure**는 Microsoft Learn 원문에 접근할 수 없어 **검색 결과 요약**으로 대조했습니다. (확인일: 2026-11-20)    
[Connect to cloud object storage using Unity Catalog](https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/external-locations) ·
[S3 storage credential & external location (manual)](https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/s3/s3-external-location-manual) ·
[Automated setup](https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/s3/s3-external-location-cfn) ·
[Manage external locations](https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/manage-external-locations) ·
[DBFS mounts (deprecated)](https://docs.databricks.com/aws/en/dbfs/mounts) ·
[Azure: ADLS Gen2 external location](https://learn.microsoft.com/en-us/azure/databricks/connect/unity-catalog/cloud-storage/external-locations-adls) ·
[Azure: managed identities](https://learn.microsoft.com/en-us/azure/databricks/connect/unity-catalog/cloud-storage/azure-managed-identities)
{: .notice--warning}

# 1. 상황

클라우드 스토리지(S3, ADLS Gen2)의 파일을 Databricks에서 읽고 써야 합니다. 예전에는 **DBFS 마운트**(`/mnt`)를 썼지만, Unity Catalog 환경에서는 **External Location**으로 접근을 **통제**하는 것이 표준입니다. 이 글은 External Location이 무엇인지와, **Azure와 AWS 각각의 등록 절차**를 정리합니다.

# 2. 한눈에 보는 요약

| 항목 | 내용 |
|---|---|
| External Location | **클라우드 스토리지 경로 + 그 경로에 접근할 수 있는 Storage Credential**을 묶은 Unity Catalog 객체 |
| Storage Credential | 클라우드 접근 인증·인가 수단. **Azure: Access Connector의 Managed Identity**, **AWS: IAM Role** |
| 접근 통제 | External Location에 `READ FILES`, `WRITE FILES`, `CREATE EXTERNAL TABLE` 등을 **GRANT/REVOKE**해서 통제 |
| 등록 순서 | **클라우드 리소스·권한 준비 → Storage Credential 생성 → External Location 생성 → 사용자에 권한 부여** |
| 마운트 | **deprecated**이며 **Unity Catalog와 호환되지 않음**. 새 워크스페이스는 마운트 기능 없이 만들어짐 |

# 3. 개념 (공식 문서)

## 3-1. 세 가지 객체의 관계

```
[클라우드 스토리지 경로]  ◀── 접근 인증 ──  [Storage Credential]
         ▲                                        ▲
         └──────────  [External Location]  ───────┘
                      (경로 + credential 묶음)
                              ▲
                  Unity Catalog 권한(GRANT)으로 사용자 통제
```

- **Storage credential**: 클라우드 테넌트의 데이터에 접근하는 **인증·인가 수단**. 예) S3는 **IAM 역할**. 어떤 사용자·그룹이 이 credential로 External Location을 정의할 수 있는지는 UC 권한이 정합니다.
- **External location**: **클라우드 스토리지 경로와, 그 경로에 대한 접근을 인가하는 storage credential을 결합**한 객체입니다. **여러 External Location이 같은 credential을 쓸 수 있습니다.** (예: credential이 `bucket/tables/*` 전체를 허용하고, External Location A와 B가 둘 다 이를 참조)
- External Location에 대한 **권한을 부여·회수**해서 해당 경로의 데이터 접근을 통제합니다. External Location 생성·사용 권한은 **외부 테이블, 외부 볼륨, 관리형 스토리지 위치가 필요한 사용자에게만** 주라고 권장합니다.

## 3-2. 어디에 쓰이나

| 용도 | 설명 |
|---|---|
| **외부 테이블·외부 볼륨** | 클라우드 스토리지에 있는 기존 데이터를 UC에 등록(포인터). 데이터 수명은 클라우드 쪽에서 관리 |
| **관리형 스토리지 위치** | 메타스토어·카탈로그·스키마의 managed storage를 지정할 때도 External Location을 참조 |
| 비정형 파일 읽기/쓰기 | 볼륨(`/Volumes/...`)으로 접근 |

- 외부 테이블/볼륨을 만들 때는 **권한이 있는 External Location에 포함된 경로**를 참조해야 합니다.
- UC는 경로 기반(URI) 접근도 지원하지만, Databricks는 **테이블은 이름으로, 볼륨은 `/Volumes` 경로로** 접근하라고 권장합니다.

## 3-3. 정리한 장점과 문서 대조

| 정리한 내용 | 공식 문서 |
|---|---|
| 스토리지 전체가 아니라 **특정 컨테이너·디렉터리 단위**로 External Location을 만들어 권한 분리 | ✅ External Location은 **경로 단위**이고, 같은 credential을 여러 External Location이 공유 가능 |
| 사용자가 **AWS Access Key나 Azure Service Principal Key를 몰라도** 됨 | ✅ 개념상 맞음. 접근은 **IAM 역할/Managed Identity** 기반 credential로 이뤄지고 사용자 권한은 UC가 통제 (직접 키를 쓰는 방식이 아님) |
| **마운트(`dbfs:/mnt`)는 보안이 취약** | ⚠️ 문서는 "취약"이 아니라 **"deprecated 패턴이며 Unity Catalog와 호환되지 않고 serverless와도 호환되지 않음"** 이라고 표현. 마운트는 **UC 거버넌스(권한·감사·계보)의 바깥**이라는 점이 핵심 (3-4) |

## 3-4. 마운트와 비교

| | DBFS 마운트 | External Location |
|---|---|---|
| 문서 상태 | **deprecated 패턴**, 새 워크스페이스에서는 **기능 자체가 제공되지 않음** | **권장 방식** |
| Unity Catalog | **마운트한 데이터는 UC와 함께 동작하지 않음** | UC 안에서 권한·감사·계보 적용 |
| serverless | **호환되지 않음** | 호환 |
| 접근 방식 | `/mnt` 아래 **별칭**. 마운트에 **스토리지 위치, 드라이버 설정, 보안 자격 증명**이 저장됨 | 경로 + credential 객체, **GRANT로 사용자별 통제** |
| 마이그레이션 | UC 거버넌스로 **옮기라고 권장** | — |

> 공식 문서는 **UC 관리형 스토리지에 UC 이외의 주체가 직접 접근하게 하지 말라**고 강하게 경고합니다. 직접 접근을 열어 두면 ① UC의 접근 통제를 우회할 수 있고 ② 감사·계보가 직접 접근을 기록하지 못하며 ③ 외부에서의 변경·삭제가 테이블을 깨뜨리거나 데이터 손상을 일으킬 수 있습니다.
{: .notice--warning}

# 4. Azure 설정 (ADLS Gen2)

전체 순서: **[Azure 인프라 설정] → [권한 부여] → [Databricks 연결]**

> 이 장의 Azure 세부 내용(역할 이름, 파일 이벤트 권한 등)은 **Microsoft Learn 원문을 읽지 못하고 검색 결과 요약으로 대조**했습니다. 실제 적용 전 최신 원문으로 확인하세요.
{: .notice--info}

## 4-1. Storage Account와 Container 만들기

1. Azure Portal에서 **Storage Account** 생성 — **Data Lake Gen2(계층 구조 네임스페이스, HNS)** 활성화 권장
2. **Container** 생성
3. 경로 형식 확인: `abfss://<컨테이너>@<스토리지계정>.dfs.core.windows.net/<경로>`

## 4-2. Access Connector 만들기

1. 리소스 그룹에서 **Access Connector for Azure Databricks** 생성 (Databricks 워크스페이스와 **같은 리전** 권장 — 리전 요건의 정확한 표현은 원문 확인 필요)
2. Access Connector는 **Managed Identity를 Databricks 계정에 연결**해 주는 Azure 1st-party 리소스입니다.
3. 나중에 쓸 **Access Connector의 리소스 ID**를 메모합니다.

## 4-3. 권한 부여 (Azure IAM)

Storage Account의 **Access control (IAM) → Add role assignment**에서, 멤버를 **Managed identity → Access Connector**로 선택해 역할을 부여합니다.

| 역할 | 대상 | 용도 |
|---|---|---|
| **Storage Blob Data Contributor** | Storage Account | **기본 접근(읽기/쓰기)** — 필수 |
| Storage Queue Data Contributor | Storage Account | **파일 이벤트**(변경 알림 구독) 사용 시 |
| Storage Account Contributor | Storage Account | 파일 이벤트용 **큐와 이벤트 라우팅 자동 생성** (없으면 큐·이벤트 구독을 **수동 구성**) |
| EventGrid EventSubscription Contributor | **Resource Group** 권장 | 파일 이벤트 **자동 설정** |

- **기본 사용만 한다면 `Storage Blob Data Contributor`만 있으면 됩니다.** 나머지 세 역할은 **파일 이벤트**(예: Auto Loader 파일 알림 모드)를 쓸 때 필요한 것입니다.
- 위 권한을 부여하려면 **Owner 또는 User Access Administrator** 역할이 필요합니다.
- 파일 이벤트 유형을 **Provided**(큐를 직접 만들어 사용)로 하면 **Storage Queue Data Contributor**만 있으면 됩니다.

> 정리해 둔 절차의 "설정 입력(Credential name, Authentication type, Access Connector ID)"은 **IAM 역할 부여 단계가 아니라 Databricks에서 credential을 만드는 단계(4-4)** 에 해당합니다. IAM에서는 **멤버로 Access Connector의 Managed Identity를 선택**합니다.
{: .notice--info}

## 4-4. Databricks에서 Credential 만들기

1. Databricks 워크스페이스 → **Catalog → Credentials** (UI 메뉴 위치는 버전에 따라 다를 수 있음)
2. **Create credential** → **Storage Credential** 선택
3. 입력
   - **Credential Type**: `Azure Managed Identity`
   - **Credential name**
   - **Access connector ID**: 만든 Access Connector의 **리소스 ID**
   - (선택) **user-assigned managed identity**를 쓰면 그 ID 입력
   - **Comment**: 어디에 쓰는 credential인지 알 수 있게 입력
4. **계정 관리자가 아닌 사용자**는 `AzureManagedIdentity` credential을 만들려면 **Access Connector에 대한 Contributor 역할**이 필요합니다.

## 4-5. External Location 만들기

1. **Catalog → External Locations → Create external location**
2. 입력
   - **External location name**
   - **URL**: 만든 컨테이너 경로 (`abfss://<컨테이너>@<계정>.dfs.core.windows.net/<경로>`)
   - **Storage credential**: 4-4에서 만든 credential
3. **Test connection**으로 검증하고, 사용자에게 권한을 부여합니다. (6장)

# 5. AWS 설정 (S3)

AWS는 두 가지 방법이 있습니다. **Databricks는 자동 설정을 대부분의 배포에 권장**하고, 많은 External Location을 만들 때 수동 방식은 권장하지 않습니다.

## 5-1. 방법 A — 자동 설정 (권장)

| 항목 | 내용 |
|---|---|
| 하는 일 | AWS 계정에 **최소 권한 IAM 역할과 신뢰 정책을 만들고**, Databricks에 **Storage Credential과 External Location**을 만들어 줌. **Auto Loader와 File Events가 자동으로 켜짐** |
| 필요 권한 | **워크스페이스 관리자 또는 메타스토어 관리자** 중 하나 |
| 방식 | **AWS IAM 임시 위임(temporary delegation)** 을 사용. 모든 자동 작업은 **CloudTrail에 기록**되고, 설정이 끝나면 **위임이 자동 만료**되어 Databricks가 계정에 상시 접근 권한을 유지하지 않음 |
| 사전 조건 | S3 버킷이 이미 존재, 버킷 이름에 **점(.) 금지**, 경로는 **ASCII만**, 버킷에 **S3 ACL 없음**, **다른 메타스토어에 등록된 경로와 겹치지 않기** |

## 5-2. 방법 B — 수동 설정

### 사전 조건 (공식 문서)

- S3 버킷을 **먼저 AWS에서 만들어 두기**. 버킷 이름에 **점 표기(`a.b.c`)를 쓰지 않기**(OpenSharing 등에서 SSL 인증서 검증 문제), 경로는 **ASCII 문자만**, **S3 ACL이 붙어 있지 않을 것**
- 다른 UC 메타스토어에 이미 External Location으로 정의된 경로 **사용 지양** (한 위치에서 **읽기**는 여러 메타스토어에서 안전하나 **동시 쓰기**는 일관성 문제)
- Databricks 권한: 메타스토어의 **`CREATE STORAGE CREDENTIAL`**, 메타스토어와 해당 credential의 **`CREATE EXTERNAL LOCATION`** (메타스토어 관리자는 기본 보유)
- AWS 권한: IAM 역할 생성용 **`iam:CreateRole`**

### Step 1. IAM 역할 만들기 (2단계로 진행)

역할은 **자기 자신을 신뢰해야(self-assuming)** 하므로, 역할이 먼저 존재해야 합니다. 그래서 먼저 **임시 신뢰 정책 + 자리표시자 external ID**로 만들고, credential을 만든 뒤 수정합니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": ["<DATABRICKS-UC-MASTER-ROLE-ARN>"] },
      "Action": "sts:AssumeRole",
      "Condition": { "StringEquals": { "sts:ExternalId": "0000" } }
    }
  ]
}
```

- `<DATABRICKS-UC-MASTER-ROLE-ARN>`은 **공식 문서에 게시된 정적인 값**(Databricks가 만든 역할)입니다. 리전·환경(일반/GovCloud)별로 값이 다르므로 **문서에서 복사**하세요.
- 이 단계에서는 **권한 정책은 아직 붙이지 않고** 역할을 저장합니다.

그다음 **같은 계정의 IAM 정책**을 만들어 역할에 붙입니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject", "s3:ListBucket",
                 "s3:GetBucketLocation", "s3:ListBucketMultipartUploads",
                 "s3:ListMultipartUploadParts", "s3:AbortMultipartUpload"],
      "Resource": ["arn:aws:s3:::<BUCKET>/*", "arn:aws:s3:::<BUCKET>"],
      "Effect": "Allow"
    },
    {
      "Action": ["kms:Decrypt", "kms:Encrypt", "kms:GenerateDataKey*"],
      "Resource": ["arn:aws:kms:<KMS-KEY>"],
      "Effect": "Allow"
    },
    {
      "Action": ["sts:AssumeRole"],
      "Resource": ["arn:aws:iam::<AWS-ACCOUNT-ID>:role/<AWS-IAM-ROLE-NAME>"],
      "Effect": "Allow"
    }
  ]
}
```

- **KMS 항목은 버킷이 암호화된 경우에만** 필요하고, 암호화를 쓰지 않으면 **KMS 섹션 전체를 지웁니다.** 암호화를 쓰면 **KMS 키 정책에도 이 역할을 허용**해야 Unity Catalog가 데이터에 접근할 수 있습니다.
- 마지막 `sts:AssumeRole` 항목이 **자기 자신(역할)** 을 가리키는 부분입니다. (self-assuming)
- 이 정책은 읽기/쓰기를 모두 허용하지만, 읽기 전용으로 쓰려면 **credential을 읽기 전용으로 표시**하면 Databricks가 IAM 역할의 쓰기 권한을 **무시**합니다.

### Step 2. Databricks에 Storage Credential 만들기

1. 메타스토어에 연결된 워크스페이스 → **Catalog → Connect → Credentials → Create credential**
2. **Credential Type: AWS IAM Role**, 이름, **IAM Role ARN**, (선택) 설명
3. (선택) **Advanced options → Limit to read-only use**로 읽기 전용 지정 (나중에 변경 가능)
4. **Create** 후 나타나는 대화상자에서 **External ID를 복사**
5. (선택) 특정 워크스페이스에만 쓰도록 **credential을 워크스페이스에 바인딩** (기본값은 메타스토어에 연결된 모든 워크스페이스에서 사용 가능)

### Step 3. IAM 역할 신뢰 정책 수정 (external ID + self-assuming)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "<DATABRICKS-UC-MASTER-ROLE-ARN>",
          "arn:aws:iam::<YOUR-AWS-ACCOUNT-ID>:role/<THIS-ROLE-NAME>"
        ]
      },
      "Action": "sts:AssumeRole",
      "Condition": { "StringEquals": { "sts:ExternalId": "<STORAGE-CREDENTIAL-EXTERNAL-ID>" } }
    }
  ]
}
```

- `0000`을 **복사한 external ID**로 바꾸고, **자기 자신(역할)의 ARN을 신뢰 주체에 추가**합니다.

### Step 4. Credential 검증

- Catalog → Connect → Credentials → 해당 credential → **Validate Configuration**
- 검증하려면 credential 소유자, 메타스토어 관리자, 또는 credential에 대한 `CREATE EXTERNAL LOCATION` 권한이 필요합니다.
- **`Self Assume Role` 검사가 실패**하면 Step 3의 신뢰 정책을 다시 확인합니다.

> **Self-assuming 강제**: AWS가 2023-06-30에 IAM 역할 신뢰 정책 요건을 바꿔서 역할이 **명시적으로 자기를 신뢰**해야 하고, Databricks는 **2025-01-20부터 self-assuming이 아닌 IAM 역할 기반 credential(기존 것 포함)의 사용을 차단**했습니다. 오래된 credential이 있다면 **검증해서 확인**하세요.
{: .notice--warning}

### Step 5. External Location 만들기

**UI**: Catalog → Connect → External Locations → Create external location → **Manual** → 이름, **Storage type: S3**, **URL**(`s3://<bucket>/<path>`), **Storage credential** 선택

**SQL**:

```sql
CREATE EXTERNAL LOCATION IF NOT EXISTS `<location-name>`
URL 's3://<bucket-path>'
WITH (STORAGE CREDENTIAL `<storage-credential-name>`)
COMMENT '<comment-string>';
```

- 이름에 하이픈 같은 **특수문자가 있으면 백틱**으로 감쌉니다.
- 특정 워크스페이스에서만 쓰게 하려면 **External Location을 워크스페이스에 바인딩**(isolation)할 수 있습니다.

# 6. 권한 부여와 운영

## 6-1. 주요 권한

| 권한 | 의미 |
|---|---|
| `READ FILES` | 경로의 파일 읽기 |
| `WRITE FILES` | 경로의 파일 쓰기. **반드시 `READ FILES`도 함께** 부여해야 함 (쓰기 전 메타데이터·경로 검증에 읽기 권한 필요. `WRITE FILES`만 있으면 `PERMISSION_DENIED`) |
| `CREATE EXTERNAL TABLE` | 이 경로를 참조하는 외부 테이블 생성 |
| `CREATE EXTERNAL VOLUME` 등 | 외부 볼륨 생성 |
| `MANAGE` | External Location 권한 관리 (또는 credential·메타스토어의 `CREATE EXTERNAL LOCATION`) |

```sql
GRANT READ FILES, WRITE FILES ON EXTERNAL LOCATION `<location-name>` TO `<principal>`;
GRANT CREATE EXTERNAL TABLE   ON EXTERNAL LOCATION `<location-name>` TO `<principal>`;
SHOW GRANTS ON EXTERNAL LOCATION `<location-name>`;
```

- `<principal>`은 **계정 수준의 사용자 이메일 또는 계정 수준 그룹 이름**입니다. 공백이나 `@`가 있으면 **백틱**으로 감쌉니다.

## 6-2. 읽기 전용과 워크스페이스 제한

- **읽기 전용 표시**: External Location(또는 credential)을 읽기 전용으로 지정하면 쓰기를 막습니다. 변경이 **모든 컴퓨트에 반영되기까지 최대 60분**이 걸릴 수 있습니다.
- **워크스페이스 제한(바인딩)**: 기본적으로 External Location은 **메타스토어에 연결된 모든 워크스페이스**에서 접근할 수 있습니다. 운영 데이터가 있는 External Location을 **운영 워크스페이스에서만** 쓰게 하는 식으로 격리할 수 있습니다.

## 6-3. 파일 이벤트 (선택)

- External Location에 **file events**를 켜면 클라우드가 보내는 **변경 알림**을 수집(예: Auto Loader 파일 알림 모드)합니다. 파일 이벤트가 **Databricks에 더 많은 데이터 접근 권한을 주는 것은 아닙니다.**
- 유형은 **Automatic(권장)**: Databricks가 알림 인프라를 만들고 검증. AWS에서는 **S3 버킷이 `csms-*` 접두사의 관리형 SNS 토픽으로 알림을 게시**하고 **SQS 큐**가 구독하며, 큐에서 **파일 경로 메타데이터만** 읽습니다(파일 내용은 지나가지 않음). 직접 관리하는 유형도 있습니다.
- Azure에서는 **큐와 Event Grid 구독**이 필요하고 4-3의 역할이 쓰입니다.

# 7. Azure와 AWS 대응표

| 항목 | Azure | AWS |
|---|---|---|
| 스토리지 | Storage Account + Container (ADLS Gen2, **HNS 활성화**) | S3 버킷 (**점 금지, ACL 없음, ASCII 경로**) |
| 경로 형식 | `abfss://<container>@<account>.dfs.core.windows.net/<path>` | `s3://<bucket>/<path>` |
| 인증 수단 | **Access Connector for Azure Databricks**의 **Managed Identity** | **IAM Role** (cross-account 신뢰 + **external ID**, **self-assuming**) |
| 스토리지 권한 | **Storage Blob Data Contributor** (IAM 역할 할당) | **IAM 정책**(S3 액션 + 필요 시 KMS) |
| Credential 유형 | `Azure Managed Identity` + Access connector ID | `AWS IAM Role` + Role ARN → **External ID 복사 → 신뢰 정책 수정** |
| 파일 이벤트 | Storage Queue + Event Grid (역할: Queue Data Contributor, Account Contributor, EventGrid EventSubscription Contributor) | **SNS 토픽(`csms-*`) + SQS 큐** |
| 자동 설정 | (이번에 확인하지 못함) | **CloudFormation 기반 자동 설정(권장)** |
| 검증 | Test connection | **Validate Configuration** + **Self Assume Role** 검사 |

# 8. 자주 만나는 문제

| 증상 | 원인·조치 |
|---|---|
| `Failed - Self Assume Role` (AWS) | 역할 신뢰 정책에 **자기 자신의 ARN**이 없음 → Step 3 수정 |
| credential이 갑자기 차단됨 (AWS) | **self-assuming이 아닌 IAM 역할** (2025-01-20부터 차단) → 신뢰 정책 수정 후 검증 |
| `WRITE FILES`만 줬는데 `PERMISSION_DENIED` | **`READ FILES`도 함께** 부여 |
| S3 암호화 버킷에 접근 불가 | **KMS 키 정책**에 역할 허용 추가, IAM 정책의 KMS 섹션 확인 |
| 버킷 이름에 점이 있음 | Databricks가 **점 표기 버킷을 지원하지 않음** → 다른 버킷 사용 |
| 경로에 한글 등 비ASCII 문자 | External Location 경로는 **ASCII만** 허용 |
| `CREATE EXTERNAL LOCATION` 권한 오류 | 메타스토어와 **credential 양쪽**의 `CREATE EXTERNAL LOCATION` 필요 |
| Azure에서 비관리자가 credential 생성 실패 | **Access Connector에 대한 Contributor 역할** 필요 |
| 권한 변경이 바로 반영되지 않음 | 읽기 전용 변경은 **최대 60분** 걸릴 수 있음 |

# 9. 체크리스트

- [ ] **Storage Credential → External Location → 권한 부여** 순서로 만들었는가?
- [ ] (Azure) IAM 역할 할당의 멤버가 **Access Connector의 Managed Identity**인가? 기본은 **Storage Blob Data Contributor**뿐이고, 파일 이벤트를 쓸 때만 나머지 역할이 필요한가?
- [ ] (AWS) 신뢰 정책에 **external ID와 자기 자신의 ARN**이 있는가? **Validate Configuration**을 통과했는가?
- [ ] (AWS) 버킷에 **점·ACL이 없고**, 경로가 **ASCII**이며, 다른 메타스토어와 **경로가 겹치지 않는가**?
- [ ] 사용자에게 **`READ FILES`와 `WRITE FILES`를 함께** 주었는가? **읽기 전용이 필요하면 read-only로 지정**했는가?
- [ ] **UC 관리형 스토리지에 직접 접근 권한**을 다른 주체에게 열어 두지 않았는가?
- [ ] **마운트(`/mnt`)를 쓰고 있다면 UC External Location으로 이전**할 계획이 있는가?

# 10. 이 글의 근거 (무엇이 문서·요약·추정인가)

| # | 주장 | 근거 유형 | 근거 내용 |
|---|---|---|---|
| 1 | External Location = 경로 + Storage Credential, 권한으로 통제 | **공식 문서(AWS)** | Connect to cloud object storage 문서 |
| 2 | 마운트는 deprecated이며 UC·serverless와 호환되지 않음 | **공식 문서(AWS)** | DBFS mounts 문서 |
| 3 | AWS 수동·자동 설정 절차와 제약(점 금지, ASCII, ACL, self-assuming) | **공식 문서(AWS)** | S3 external location 문서 |
| 4 | `WRITE FILES`는 `READ FILES`가 필요 | **공식 문서(AWS)** | Manage external locations |
| 5 | Azure의 Access Connector, 역할(Blob/Queue/Account/EventGrid)과 파일 이벤트 요건 | **검색 요약(공식 원문 미확인)** | Microsoft Learn 검색 결과 요약. **원문 미확인** |
| 6 | Access Connector를 워크스페이스와 같은 리전에 둔다 | **사용자 정리 내용** | 원문 미확인 |
| 7 | 마운트는 "보안이 취약하다" | **문서에 없는 표현** | 문서의 표현은 deprecated, UC 비호환 |
| 8 | 코드·설정 예시 | **예시** | 직접 실행해 검증하지 않음 (**직접 테스트 결과 없음**) |

- 이 글은 **직접 테스트한 결과가 없는 설정 절차 정리**여서 포트폴리오 자료는 만들지 않았습니다.

# 11. 한 줄 정리

- **External Location = 클라우드 경로 + Storage Credential**이고, **GRANT로 접근을 통제**한다. 마운트는 **deprecated이며 UC와 호환되지 않는다.**
- **Azure는 Access Connector(Managed Identity) + `Storage Blob Data Contributor`**, **AWS는 IAM 역할(+ external ID, self-assuming)** 으로 credential을 만든다.
- **WRITE FILES에는 READ FILES가 필요**하고, AWS는 **자동 설정이 권장**되며 **self-assuming 역할이 필수**다.

# 관련 용어 글
External Location · Storage Credential · DBFS 마운트 · Unity Catalog 테이블 권한

## 🔗 참고
- [Connect to cloud object storage using Unity Catalog](https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/external-locations)
- [Create a storage credential and external location for S3](https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/s3/s3-external-location-manual)
- [Create an external location with automated setup](https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/s3/s3-external-location-cfn)
- [Manage external locations](https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/manage-external-locations)
- [Mount object storage to DBFS (legacy)](https://docs.databricks.com/aws/en/dbfs/mounts)
- [Azure: Connect to an ADLS Gen2 external location](https://learn.microsoft.com/en-us/azure/databricks/connect/unity-catalog/cloud-storage/external-locations-adls)
- [Azure: Use Azure managed identities in Unity Catalog](https://learn.microsoft.com/en-us/azure/databricks/connect/unity-catalog/cloud-storage/azure-managed-identities)
