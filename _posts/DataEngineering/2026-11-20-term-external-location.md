---
title:  "[DE/용어] External Location"
excerpt: "클라우드 스토리지 경로와 접근 자격을 묶어 Unity Catalog 권한으로 통제하는 객체"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, Unity Catalog, External Location, Governance]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[Connect to cloud object storage using Unity Catalog](https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/external-locations)    
[Manage external locations](https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/manage-external-locations)
{: .notice--warning}

# 0. 이 글을 읽기 전에

Databricks에서 S3나 ADLS의 파일에 접근하려면 "누가 어느 경로를 읽고 쓸 수 있는가"를 정해야 합니다. Unity Catalog는 이를 **External Location**이라는 객체로 통제합니다.

# 1. 한 줄 정의

**External Location**은 **클라우드 스토리지 경로**와 **그 경로에 대한 접근을 인가하는 Storage Credential**을 결합한 Unity Catalog 객체입니다. 이 객체에 **권한을 부여·회수**해서 해당 경로의 데이터 접근을 통제합니다.

```
External Location = 경로(s3://bucket/path, abfss://...) + Storage Credential
        └─ GRANT READ FILES / WRITE FILES / CREATE EXTERNAL TABLE ... 로 사용자 통제
```

# 2. 핵심 개념

| 항목 | 내용 |
|---|---|
| 경로 단위 | 버킷·컨테이너 전체가 아니라 **특정 경로**에 대해 만들 수 있어 **권한을 분리**할 수 있음 |
| credential 공유 | **여러 External Location이 같은 Storage Credential**을 쓸 수 있음 |
| 용도 | **외부 테이블·외부 볼륨**, **관리형 스토리지 위치**(메타스토어·카탈로그·스키마) |
| 접근 범위 | 기본적으로 메타스토어에 연결된 **모든 워크스페이스**에서 접근 가능. **워크스페이스에 바인딩**해서 제한 가능 |
| 읽기 전용 | 읽기 전용으로 표시 가능(반영에 최대 60분) |
| 파일 이벤트 | 켜면 클라우드 변경 알림을 수집(예: Auto Loader). Databricks에 더 많은 데이터 접근을 주지는 않음 |

# 3. 권한

| 권한 | 의미 |
|---|---|
| `READ FILES` | 파일 읽기 |
| `WRITE FILES` | 파일 쓰기 (**`READ FILES`도 함께 필요**) |
| `CREATE EXTERNAL TABLE` | 외부 테이블 생성 |
| `CREATE EXTERNAL VOLUME` | 외부 볼륨 생성 |
| `MANAGE` | 권한 관리 |

```sql
CREATE EXTERNAL LOCATION IF NOT EXISTS `sales_landing`
URL 's3://example-bucket/landing/sales'
WITH (STORAGE CREDENTIAL `sales_cred`);

GRANT READ FILES, WRITE FILES ON EXTERNAL LOCATION `sales_landing` TO `data_eng`;
SHOW GRANTS ON EXTERNAL LOCATION `sales_landing`;
```

- 부여 대상은 **계정 수준 사용자 또는 그룹**입니다.

# 4. 사용 원칙 (공식 문서)

- External Location 생성·사용 권한은 **외부 테이블, 외부 볼륨, 관리형 스토리지 위치가 필요한 사용자에게만** 줍니다.
- 테이블은 **이름**으로, 볼륨은 **`/Volumes` 경로**로 접근하는 것을 권장하고, 경로(URI) 직접 접근은 피합니다.
- **UC 관리형 스토리지에 UC 이외의 주체가 직접 접근하게 하지 않습니다.** (접근 통제 우회, 감사·계보 누락, 데이터 손상 위험)

# 5. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "External Location은 스토리지 자체다" | **경로 + credential을 묶은 UC 객체**. 데이터는 클라우드에 있음 |
| "만들면 모두 접근할 수 있다" | **GRANT를 받은 사용자만** |
| "WRITE FILES만 주면 쓸 수 있다" | **READ FILES도 필요** |
| "외부 테이블에만 쓰인다" | **관리형 스토리지 위치**를 지정할 때도 참조 |

# 6. 한 줄 정리

- External Location = **경로 + Storage Credential**, **GRANT로 접근 통제**. 외부 테이블·볼륨과 관리형 스토리지에 쓰인다.
- **WRITE FILES에는 READ FILES가 필요**하다.

# 관련 용어
Storage Credential, DBFS 마운트, Unity Catalog 테이블 권한

## 🔗 참고
- [Connect to cloud object storage using Unity Catalog](https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/external-locations)
- [Manage external locations](https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/manage-external-locations)
