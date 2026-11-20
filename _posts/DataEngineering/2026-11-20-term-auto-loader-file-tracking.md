---
title:  "[DE/용어] Auto Loader 파일 추적과 체크포인트"
excerpt: "Auto Loader가 '이미 처리한 파일'을 기억하는 방법과 체크포인트를 다룰 때의 규칙"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, Auto Loader, Checkpoint, RocksDB]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[What is Auto Loader?](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/)    
[Configure Auto Loader for production workloads](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/production)    
[Auto Loader FAQ](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/faq)
{: .notice--warning}

# 0. 이 글을 읽기 전에

**Auto Loader** 글에서 "새 파일만 읽는다"고 했습니다. 그렇다면 Auto Loader는 **어떤 파일을 이미 읽었는지 어디에 어떻게 기억**할까요? 이 글은 그 저장 방식과 **체크포인트를 건드릴 때 지켜야 할 규칙**을 설명합니다.

# 1. 한 줄 정의

Auto Loader는 파일을 발견할 때마다 **그 파일의 메타데이터를 체크포인트 위치의 RocksDB(키-값 저장소)에 저장**해서, 각 파일이 **정확히 한 번만** 처리되도록 보장합니다. 이 저장된 정보를 **파일 이벤트 상태(file event state)** 라고 부를 수 있습니다.

# 2. 쉬운 비유 (이해를 돕기 위한 비유)

도서관 **반납 장부**를 떠올리면 됩니다. 새 책(파일)이 들어오면 장부에 한 줄을 적고, 이미 적힌 책은 다시 처리하지 않습니다. 장부(=체크포인트)를 **잃어버리거나 다른 장부로 바꾸면** 어떤 책이 처리됐는지 모르게 되어 처음부터 다시 하게 됩니다.

# 3. 무엇이 어디에 저장되나

```
체크포인트 위치(checkpointLocation)
 ├─ (스트림 진행 상황: 어디까지 처리했는지)
 └─ 파일 이벤트 상태 (RocksDB)   ← 발견한 파일의 메타데이터
      path, size, create_time, discovery_time, commit_time ...
```

- 이 상태는 `cloud_files_state` 함수로 **테이블처럼 조회**할 수 있습니다. (→ cloud_files_state 글)
- 위 폴더 구조는 이해를 위한 개념도이며 실제 내부 구성은 다를 수 있습니다.

# 4. 파일을 "같은 파일"로 판단하는 기준

공식 FAQ:
- 기본적으로 **파일 경로**를 기준으로 **한 번만** 수집합니다.
- `cloudFiles.allowOverwrites = true`이면 **마지막 수정 시각**도 함께 보고, 새 파일인지 **갱신된 파일**인지 판단해 **다시 수집**합니다.
- 기본값(`false`)에서 파일이 **덮어쓰기/추가**되면 **어느 버전이 처리될지 보장할 수 없습니다.** → 문서는 **불변 파일만 수집**하라고 권장합니다.

# 5. 체크포인트 규칙

| 규칙 | 이유 |
|---|---|
| 체크포인트 위치에 **클라우드 객체 수명 주기 정책을 걸지 않는다** | 정책이 파일을 정리하면 **스트림 상태가 손상**되고 **처음부터 다시 시작**해야 함 |
| 체크포인트 위치를 **바꾸지 않는다** | 위치를 바꾸면 **이전 스트림을 버리고 새 스트림을 시작**한 것과 같음 (이미 처리한 파일을 다시 처리할 수 있음) |
| 스트림마다 **서로 다른 체크포인트**를 쓴다 | 체크포인트는 **스트림의 식별 정보**를 담고 있음 |
| 고용량·장기 스트림은 **DBR 15.4 LTS 이상** | RocksDB 상태 전체를 내려받을 때까지 기다리지 않고 시작해 **시작이 빨라짐** |

> 수명 주기 정책은 `MOVE`로 보관한 **아카이브 폴더**에는 걸 수 있지만(예: 저비용 계층으로 이동), **체크포인트 폴더**에는 걸지 마세요.
{: .notice--warning}

# 6. 상태는 계속 커진다

- 파일이 쌓일수록 RocksDB의 상태도 **계속 커집니다.** 파일이 수백만 개 이상이 되면 시작 시간과 성능에 영향이 있을 수 있습니다.
- **상태 크기를 제어**하는 방법은 두 가지입니다.
  1. **`cloudFiles.maxFileAge`**: 일정 기간이 지난 **파일 이벤트를 만료**시킴 (→ maxFileAge 글, 위험 있음)
  2. **`cloudFiles.cleanSource`**: 처리된 **원본 파일을 이동·삭제**해서 파일 수 자체를 줄임 (→ cleanSource 글)
- 만료된 이벤트는 RocksDB에 **삭제 표시(tombstone)** 로 남아서 저장소 사용량이 **일시적으로 늘었다가 안정**될 수 있습니다.

# 7. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "체크포인트는 임시 폴더다. 지워도 된다" | 지우면 **파일 처리 이력을 잃어** 처음부터 다시 처리할 수 있음 |
| "로그 정리 정책을 전체 버킷에 걸었다" | **체크포인트 폴더가 포함되면 상태 손상** |
| "경로가 같으면 체크포인트도 같아도 된다" | 스트림마다 따로. 위치 변경은 **새 스트림** |
| "exactly-once니까 어떤 sink든 중복이 없다" | Delta Lake에 쓸 때 보장됨. `foreachBatch` 같은 **멱등하지 않은 sink**는 재처리될 수 있음 (→ 멱등성 글) |

# 8. 한 줄 정리

- Auto Loader는 **체크포인트의 RocksDB에 파일별 메타데이터를 저장**해 **정확히 한 번** 처리한다. 기본 판단 기준은 **파일 경로**.
- 체크포인트에는 **수명 주기 정책을 걸지 말고, 위치를 바꾸지 말 것**. 지키지 않으면 **상태 손상/재처리**.
- 상태 크기는 **`maxFileAge`(위험)** 와 **`cleanSource`(원본 정리)** 로 관리하고, 고용량은 **DBR 15.4 LTS+** 를 쓴다.

# 관련 용어
Auto Loader, cloud_files_state, cloudFiles.maxFileAge, cloudFiles.cleanSource, 멱등성

## 🔗 참고
- [What is Auto Loader?](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/)
- [Configure Auto Loader for production workloads](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/production)
- [Auto Loader FAQ](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/faq)
