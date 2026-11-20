---
title:  "[DE/용어] Auto Loader"
excerpt: "클라우드 스토리지에 도착하는 새 파일을 자동으로, 증분으로 수집하는 Databricks의 수집 방식"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, Auto Loader, File Ingestion, Structured Streaming]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[What is Auto Loader?](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/)    
[Auto Loader file detection modes](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/file-detection-modes)    
[Auto Loader options](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/options)    
[Auto Loader FAQ](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/faq)
{: .notice--warning}

# 0. 이 글을 읽기 전에

"스토리지에 파일이 계속 떨어지는데, 어제까지 읽은 파일은 건너뛰고 **새 파일만** 읽어서 테이블에 넣고 싶다"는 요구에 쓰는 기능입니다. **Structured Streaming**(스트리밍으로 데이터를 조금씩 읽는 Spark의 방식)을 알고 있으면 쉽지만, 모르셔도 괜찮게 설명합니다.

# 1. 한 줄 정의

**Auto Loader**는 클라우드 스토리지에 도착하는 **새 데이터 파일을 추가 설정 없이 증분(incremental)으로 효율적으로 처리**하는 기능입니다. Structured Streaming 소스 이름은 **`cloudFiles`** 입니다. 입력 디렉터리 경로를 주면 새 파일을 자동으로 처리하고, 원하면 **기존 파일도 함께** 처리합니다.

# 2. 왜 필요한가

일반 배치 읽기(`spark.read.parquet("경로")`)는 매번 **폴더의 모든 파일**을 읽습니다. 그러면 다음 문제가 생깁니다.

| 문제 | 일반 배치 읽기 | Auto Loader |
|---|---|---|
| 어제 읽은 파일을 또 읽음 | 직접 구분해야 함(날짜 폴더, 처리 목록 관리 등) | **알아서 건너뜀** |
| 파일이 수백만 개 | 목록 조회만으로도 오래 걸림 | 증분 발견으로 확장 |
| 중간에 실패 | 어디까지 읽었는지 직접 관리 | 체크포인트로 **이어서 처리** |
| 같은 파일이 두 번 들어감 | 직접 중복 제거 | **exactly-once** 보장 |

공식 문서: 수십억 개 파일의 마이그레이션·백필에도 쓸 수 있고, **시간당 수백만 파일**의 준실시간 수집까지 확장됩니다.

# 3. 가장 단순한 사용 예

```python
df = (spark.readStream.format("cloudFiles")
      .option("cloudFiles.format", "json")                  # 필수: 원본 파일 형식
      .option("cloudFiles.schemaLocation", "/Volumes/main/landing/_schemas/orders")
      .load("s3://example-bucket/landing/orders/"))         # 원본 폴더

(df.writeStream
   .option("checkpointLocation", "/Volumes/main/landing/_checkpoints/orders")
   .trigger(availableNow=True)                              # 지금 있는 파일을 모두 처리하고 종료
   .toTable("main.bronze.orders_raw"))
```

- `cloudFiles.format`은 **필수 옵션**입니다 (`avro`, `binaryFile`, `csv`, `json`, `orc`, `parquet`, `text`, `xml`).
- **체크포인트**(`checkpointLocation`)에 "어떤 파일까지 읽었는지"가 저장됩니다. (→ Auto Loader 파일 추적과 체크포인트 글)
- Lakeflow 파이프라인에서 쓰면 **스키마와 체크포인트 위치를 파이프라인이 자동 관리**하므로 직접 지정하지 않아도 됩니다. 공식 문서는 증분 수집에 **Lakeflow 파이프라인의 Auto Loader를 권장**합니다.

# 4. 읽을 수 있는 곳과 형식

| 구분 | 내용 |
|---|---|
| 스토리지 | Amazon S3(`s3://`), ADLS(`abfss://`), GCS(`gs://`), **Unity Catalog 볼륨**(`/Volumes/`), Azure Blob(`wasbs://`, 레거시 WASB 드라이버는 **deprecated**) |
| 파일 형식 | JSON, CSV, XML, PARQUET, AVRO, ORC, TEXT, BINARYFILE (지원하는 **압축** 형식도 읽음) |
| 베타 | 파일을 **FILE 참조**로 수집(문서·이미지 등) |

# 5. 새 파일을 어떻게 발견하나: 파일 감지 모드

Auto Loader에는 새 파일을 알아내는 방법이 두 가지 있습니다.

| 모드 | 동작 | 특징 |
|---|---|---|
| **디렉터리 리스팅**(기본) | 입력 디렉터리를 **목록 조회**해서 새 파일을 찾음 | **추가 권한 설정 없이** 빠르게 시작. DBR 9.1+에서는 파일이 **사전순으로 도착하는지 자동 감지**해서 API 호출을 크게 줄임 |
| **파일 알림**(권장) | 클라우드의 **알림·큐 서비스**가 파일 이벤트를 구독 | 더 빠르고 확장성이 좋음. 외부 위치(external location)에서 **file events**를 켜면 **추가 권한 설정 불필요** |

- 문서는 **대부분의 워크로드에 "파일 이벤트 기반 파일 알림 모드"** 를 권장하고, 지금 디렉터리 리스팅을 쓰고 있다면 **전환을 권장**합니다.
- 모드를 **재시작 사이에 바꿔도 exactly-once가 유지**됩니다. 볼륨으로 이전해도 마찬가지입니다.
- **파일 이벤트 기반 알림**의 지원 범위: S3·ADLS·GCS는 **DBR 14.3 LTS 이상**, Azure Blob Storage는 **미지원**.
- ⚠️ **어떤 모드에서도 파일을 발견·처리하는 순서는 보장되지 않습니다.** 순서가 뒤바뀐 도착을 견디게 설계하세요.
- 알림은 **최소 한 번** 전달되지만 파일이 **누락될 수 있어서**, `cloudFiles.backfillInterval`(예: `1 day`)로 **주기적 백필(비동기 디렉터리 리스팅)** 을 돌려 놓친 파일을 찾을 수 있습니다. 이 간격은 **기본값이 없고**, 디렉터리 리스팅과 파일 이벤트 없는 알림 모드에서는 **설정해야만** 백필이 돕니다.

# 6. 자주 쓰는 옵션

| 옵션 | 기본값 | 설명 |
|---|---|---|
| `cloudFiles.format` | 없음(필수) | 원본 파일 형식 |
| `cloudFiles.includeExistingFiles` | `true` | 입력 경로의 **기존 파일도** 처리할지. **스트림을 처음 시작할 때만** 평가됨 |
| `cloudFiles.maxFilesPerTrigger` | `1000` | 트리거당 처리할 **최대 파일 수**(하드 한도). DBR 18.0+에서는 **동적으로 설정**됨 |
| `cloudFiles.maxBytesPerTrigger` | 없음 | 트리거당 최대 바이트(**소프트 한도**). 파일 하나는 **쪼개지 않음** |
| `cloudFiles.allowOverwrites` | `false` | 입력 파일이 **덮어쓰기/추가**됐을 때 다시 처리할지 |
| `cloudFiles.maxFileAge` | 없음 | 파일 이벤트 추적 기간 (→ maxFileAge 글) |
| `cloudFiles.cleanSource` | `OFF` | 처리된 파일 이동/삭제 (→ cleanSource 글) |

- `Trigger.AvailableNow`는 **쿼리 시작 시점 이전에 도착한 파일을 모두 처리**하고, 시작 후 도착한 파일은 **다음 트리거까지 무시**합니다. Lakeflow Jobs에서 **배치처럼 예약 실행**할 때 씁니다.
- `allowOverwrites=true`는 **조심해서** 써야 합니다. 파일이 수정되면 **파일 전체를 다시 처리**하므로 **중복 레코드를 직접 처리**해야 하고, 파일 알림 모드에서는 **같은 파일이 두 번 수집**될 수도 있습니다. 문서는 **불변 파일만 수집하고 기본값을 쓰라**고 권장합니다.

# 7. 자주 헷갈리는 점

| 오해 | 실제 |
|---|---|
| "스트리밍이니까 항상 켜 둬야 한다" | `availableNow` 트리거로 **배치처럼 실행하고 끝낼 수 있음** |
| "파일 순서대로 처리된다" | **순서 보장 없음** |
| "파일이 수정돼도 알아서 다시 읽는다" | 기본은 **경로 기준 1회 처리**. 수정 반영은 `allowOverwrites` 필요(권장하지 않음) |
| "체크포인트는 아무 데나 둬도 된다" | 위치를 **바꾸면 새 스트림**, **수명 주기 정책이 걸리면 상태 손상** |
| "처음 시작할 때 `includeExistingFiles=false`면 폴더를 안 읽는다" | 이후 생성된 파일을 찾기 위해 **리스팅은 수행**함 |

# 8. 한 줄 정리

- Auto Loader = **새 파일만 자동·증분으로 읽는 `cloudFiles` 스트리밍 소스**. 체크포인트의 상태로 **exactly-once**를 보장한다.
- 파일 감지는 **디렉터리 리스팅(기본)** 과 **파일 알림(권장, file events)** 이 있고, **순서는 보장되지 않는다.**
- 배치처럼 쓰려면 **`availableNow`** 트리거, 상태 관리는 **체크포인트/`maxFileAge`/`cleanSource`**.

# 관련 용어
Auto Loader 파일 추적과 체크포인트, cloud_files_state, cloudFiles.maxFileAge, cloudFiles.cleanSource, 멱등성

## 🔗 참고
- [What is Auto Loader?](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/)
- [Auto Loader file detection modes](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/file-detection-modes)
- [Auto Loader options](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/options)
- [Auto Loader FAQ](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/faq)
- [Configure Auto Loader for production workloads](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/production)
