---
title:  "[DE/용어] cloudFiles.maxFileAge"
excerpt: "파일 이벤트 추적 기간을 제한해 체크포인트 상태 증가를 막는 옵션과 그 위험"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, Auto Loader, maxFileAge, State Management]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[Auto Loader options](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/options)    
[Configure Auto Loader for production workloads](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/production)
{: .notice--warning}

# 0. 이 글을 읽기 전에

**Auto Loader 파일 추적과 체크포인트** 글에서 파일 상태가 계속 쌓인다고 했습니다. 이 글은 그 증가를 막는 `cloudFiles.maxFileAge`와, **왜 함부로 건드리면 안 되는지**를 설명합니다.

# 1. 한 줄 정의

`cloudFiles.maxFileAge`는 **파일 이벤트를 중복 제거 목적으로 추적하는 기간**입니다. 이 기간이 지난 파일 이벤트는 체크포인트에서 **만료(삭제)** 됩니다. "이 기간보다 오래된 파일은 이미 처리된 지 오래니, 더 이상 중복 체크를 위해 기억하지 않겠다"는 뜻입니다.

# 2. 공식 문서의 핵심

| 항목 | 내용 |
|---|---|
| 기본값 | **없음(None)** |
| 값 형식 | 기간 문자열 (예: `90 days`) |
| **최소값** | **14일** |
| 성격 | 고용량 데이터셋의 **비용 제어 장치** |
| 권고 | **시간당 수백만 파일** 규모가 아니면 **조정하지 말 것** |
| 조정한다면 | **보수적으로**, 예: **90일** ("유사한 수집 솔루션이 권장하는 수준") |

# 3. 위험: 중복과 누락 둘 다 생긴다

공식 경고: 너무 공격적으로 줄이면 **중복 적재 또는 파일 누락**이 생길 수 있습니다.

| 상황 | 결과 |
|---|---|
| 처리한 파일의 기록이 **만료**된 뒤 그 파일이 다시 보임 | **재처리 → 중복 적재** |
| 스트림이 **오래 중단**됐다가 재시작 (파일 알림 모드) | 큐에서 가져온 이벤트 중 `maxFileAge`보다 **오래된 것은 무시** → **누락** |
| 스트림이 오래 중단됐다가 재시작 (디렉터리 리스팅 모드) | 중단 중 생겼지만 `maxFileAge`보다 **오래된 파일은 무시** → **누락** |
| 디렉터리 리스팅 모드에서 `1 month` → `2 months`로 **늘려서** 재시작 | 1개월보다 오래되고 2개월보다 최근인 파일이 **재처리** |
| **처음 시작할 때** 설정 | `maxFileAge`보다 오래된 **기존 데이터는 수집되지 않음** |

## 예시로 이해하기

`maxFileAge = 30 days`라고 하고, 스트림이 **45일 동안 멈췄다가** 재시작한다고 해 봅시다.

```
45일 전  ──────────── 30일 전 ───────────── 오늘
         (멈추기 전)    [maxFileAge 한계]
  파일 A(40일 전 도착): 중단 중 생겼지만 30일보다 오래됨 → 무시 → 영영 수집 안 될 수 있음 (누락)
```

처음부터 오래된 데이터까지 적재해야 한다면 **첫 실행에서는 이 옵션을 쓰지 말고, 이후 실행부터** 설정하세요.

# 4. 만료의 부작용

만료된 이벤트는 RocksDB에서 **삭제 표시(tombstone)** 로 남습니다. 그래서 **저장소 사용량이 일시적으로 증가했다가** 안정화될 수 있습니다.

# 5. 언제 쓰나 / 쓰지 않나

| 상황 | 판단 |
|---|---|
| 파일이 **수백만 개/시간** 수준으로 들어오고 체크포인트가 무거움 | 고려 (먼저 **DBR 15.4 LTS+**, **파일 이벤트 모드**를 확인) |
| 일반적인 규모 | **설정하지 않음** (문서 권고) |
| 장기 중단 후 재시작이 **있을 수 있음** | 누락 위험 때문에 **더 신중히** |
| 원본 파일을 정리(`cleanSource`)하고 있음 | 파일 수 자체가 줄어드니 **먼저 cleanSource**를 검토 |

# 6. 설정 예

```python
df = (spark.readStream.format("cloudFiles")
      .option("cloudFiles.format", "parquet")
      .option("cloudFiles.maxFileAge", "90 days")      # 최소 14일, 보수적으로
      .load("s3://example-bucket/landing/orders/"))
```

# 7. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "줄이면 성능이 좋아진다" | 상태는 줄지만 **중복/누락 위험**이 따름. 비용 제어용 |
| "위험은 중복뿐이다" | **누락**도 있음 |
| "아무 값이나 설정 가능" | **최소 14일** |
| "처음부터 설정하는 게 좋다" | **오래된 데이터를 적재하려면 처음엔 설정 금지** |

# 8. 한 줄 정리

- `maxFileAge` = **파일 이벤트 추적 기간**. 지나면 체크포인트에서 **만료**. 기본 없음, **최소 14일**.
- **중복과 누락 모두** 생길 수 있어 문서는 **수백만 파일/시간이 아니면 조정하지 말 것**, 한다면 **90일 같은 보수적 값**을 권한다.
- 상태 증가는 **DBR 15.4+**, **파일 이벤트 모드**, **`cleanSource`** 를 먼저 검토한다.

# 관련 용어
Auto Loader 파일 추적과 체크포인트, cloudFiles.cleanSource, cloud_files_state

## 🔗 참고
- [Auto Loader options](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/options)
- [Configure Auto Loader for production workloads](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/production)
