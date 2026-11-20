---
title:  "[DE/용어] DBFS 마운트 (/mnt)"
excerpt: "클라우드 스토리지를 /mnt 경로의 별칭으로 연결하던 방식과, 왜 External Location으로 옮기라고 하는지"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, DBFS, Mount, Legacy]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[Mount object storage to DBFS (legacy)](https://docs.databricks.com/aws/en/dbfs/mounts)
{: .notice--warning}

# 0. 이 글을 읽기 전에

예전 노트북에서 `dbfs:/mnt/...` 경로를 본 적이 있다면 **마운트**입니다. 지금은 **쓰지 말라고 권장되는** 방식이므로 배경과 대안을 이해해 두세요.

# 1. 한 줄 정의

**DBFS 마운트**는 클라우드 오브젝트 스토리지를 **DBFS(Databricks File System)의 `/mnt` 아래 별칭**으로 연결해서, 클라우드에 익숙하지 않은 사용자도 **익숙한 파일 경로**로 데이터를 쓰게 해 주던 기능입니다.

# 2. 마운트가 저장하는 것

- 클라우드 오브젝트 스토리지의 **위치**
- 스토리지 계정/컨테이너에 연결하는 **드라이버 설정**
- 데이터에 접근하는 데 필요한 **보안 자격 증명**

```python
# (예전 방식, 신규 사용 비권장)
dbutils.fs.mount(source="s3a://example-bucket/landing", mount_point="/mnt/landing")
df = spark.read.parquet("/mnt/landing/sales")
```

# 3. 공식 문서의 현재 입장

> DBFS 마운트는 **deprecated 패턴**이며 **Databricks serverless 컴퓨트 아키텍처와 호환되지 않습니다.** **새 워크스페이스는 이 기능 없이 프로비저닝**됩니다. Databricks는 **Unity Catalog External Location 사용을 권장**합니다. 마운트한 데이터는 **Unity Catalog와 함께 동작하지 않습니다.**

| 문제 | 설명 |
|---|---|
| **UC 거버넌스 밖** | 마운트한 데이터는 UC와 함께 동작하지 않아 **UC의 권한·감사·계보 모델을 받을 수 없음** |
| **serverless 비호환** | serverless 컴퓨트에서 사용할 수 없음 |
| **신규 워크스페이스 미제공** | 새 워크스페이스에는 아예 기능이 없음 |

- "보안이 취약하다"고 단정하기보다, **UC의 중앙 통제 바깥에서 동작하는 레거시 방식**이라는 점이 문서가 말하는 핵심입니다.

# 4. 마운트 → External Location 이전

| 마운트 | External Location |
|---|---|
| `/mnt/...` 별칭 | `s3://...`/`abfss://...` 경로를 External Location으로 등록 |
| 마운트에 **자격 증명이 포함** | **Storage Credential**(IAM 역할/Managed Identity)로 분리 |
| 접근 통제 없음(워크스페이스 단위) | **READ FILES/WRITE FILES를 사용자별로 GRANT** |
| 경로 직접 접근 | 테이블은 **이름**, 파일은 **`/Volumes`** 로 접근 |

# 5. 한 줄 정리

- DBFS 마운트 = `/mnt` 아래 별칭으로 클라우드 스토리지를 연결하는 **레거시 방식**. **UC·serverless와 호환되지 않고 새 워크스페이스에서는 제공되지 않는다.**
- 대안은 **Unity Catalog External Location**(+ 볼륨)이다.

# 관련 용어
External Location, Storage Credential

## 🔗 참고
- [Mount object storage to DBFS (legacy)](https://docs.databricks.com/aws/en/dbfs/mounts)
