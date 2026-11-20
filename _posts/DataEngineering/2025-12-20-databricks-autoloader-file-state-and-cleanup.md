---
title:  "[DE/Databricks] Auto Loader 파일 상태 확인과 원본 정리: cloud_files_state, maxFileAge, cleanSource"
excerpt: "처리된 파일을 확인하는 방법, 체크포인트 상태가 커지는 문제, 원본 파일 정리(MOVE/DELETE)를 공식 문서 기준으로 정리하고 흔한 오해를 바로잡기"

categories:
  - DataEngineering
tags:
  - [Databricks, Auto Loader, cloud_files_state, cleanSource, maxFileAge, Checkpoint]

toc: true
toc_sticky: true
published: true

date: 2025-12-20
last_modified_at: 2026-11-20
---

업무에서 겪은 상황을 일반화해서 정리했습니다. 경로·테이블명은 모두 **예시**입니다.    
공식 문서(AWS 기준)를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[cloud_files_state](https://docs.databricks.com/aws/en/sql/language-manual/functions/cloud_files_state) ·
[Auto Loader options](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/options) ·
[Clean up processed files](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/clean-source) ·
[Production 설정](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/production) ·
[Auto Loader 개요](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/) ·
[파일 감지 모드](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/file-detection-modes) ·
[FAQ](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/faq)
{: .notice--warning}

# 1. 상황

클라우드 스토리지의 랜딩 폴더에 매일 파일이 도착하고, **Auto Loader**로 Delta 테이블에 적재합니다. 운영하다 보니 세 가지가 궁금해집니다.

| # | 궁금한 것 | 이 글의 답 |
|---|---|---|
| ① | "이 파일은 처리됐나? 누락된 파일은 없나?" | `cloud_files_state` 함수로 조회 (4장) |
| ② | "파일이 몇백만 개 쌓이면 체크포인트가 무거워지지 않나?" | `cloudFiles.maxFileAge` (5장) |
| ③ | "처리 끝난 원본 파일을 지우거나 옮기고 싶다" | `cloudFiles.cleanSource` (6장) |

이 주제를 정리한 글이나 AI 답변에는 **공식 문서와 다른 내용이 섞여 있는 경우**가 있어서, 공식 문서와 대조한 결과를 함께 적었습니다. (8장 요약표)

# 2. 한눈에 보는 요약

| 항목 | 핵심 |
|---|---|
| 파일 추적 | 체크포인트 안의 **RocksDB**에 파일 메타데이터를 저장해 **정확히 한 번(exactly-once)** 처리 |
| 상태 조회 | `cloud_files_state('<체크포인트 경로>')` (DBR 11.3 LTS+). 일부 컬럼은 **DBR 16.4+/조건부** 채워짐 |
| 처리 완료 판정 | `commit_time`이 NULL이 아니고 `ingestion_state = 'INGESTED'` |
| 상태 크기 제어 | `cloudFiles.maxFileAge` — **최소 14일**, 문서 권장 예시 90일. 공격적으로 줄이면 **중복 적재 또는 파일 누락** |
| 원본 정리 | `cloudFiles.cleanSource` = `OFF`(기본) / `DELETE` / `MOVE` (DBR 16.4+) |
| MOVE 목적지 | **같은 external location/볼륨/마운트 안**, 소스의 **하위 경로 금지**. **다른 버킷은 미지원(오류)** |

# 3. 동작 원리: Auto Loader는 어떻게 "처리한 파일"을 기억하나

- Auto Loader는 새 파일을 발견하면 그 **메타데이터를 체크포인트 위치의 RocksDB(키-값 저장소)에 저장**합니다. 이 저장소가 "각 파일을 정확히 한 번만 처리"하는 것을 보장합니다. 실패 후에는 체크포인트의 정보로 **중단된 지점부터 이어서** 처리하고, Delta Lake에 쓸 때 exactly-once를 계속 보장합니다.
- 파일 식별은 기본적으로 **파일 경로** 기준입니다. `cloudFiles.allowOverwrites = true`이면 **마지막 수정 시각**도 함께 보고 갱신된 파일을 다시 처리합니다. 문서는 **불변(immutable) 파일만 수집**하고 기본값(`false`)을 쓰라고 권장합니다.
- 체크포인트 위치에는 **클라우드 객체 수명 주기(lifecycle) 정책을 적용하지 않는 것**을 권장합니다. 정책으로 체크포인트 파일이 정리되면 **스트림 상태가 손상**되고 **처음부터 다시 시작**해야 합니다.
- **체크포인트 위치를 바꾸면** 이전 스트림을 버리고 **새 스트림을 시작**한 것과 같습니다.
- 고용량·장기 실행 스트림은 **DBR 15.4 LTS 이상**을 권장합니다. 이 버전부터는 RocksDB 상태 전체를 내려받을 때까지 기다리지 않고 스트림이 시작돼서 **시작 시간이 빨라집니다.**
- Auto Loader는 **파일 처리 순서를 보장하지 않습니다.** 파이프라인은 순서가 뒤바뀐 도착을 견디도록 설계해야 합니다.

# 4. `cloud_files_state`로 파일 상태 조회하기

## 4-1. 문법

```sql
cloud_files_state( { TABLE ( table_name ) | checkpoint } )
```

| 인자 | 설명 |
|---|---|
| `checkpoint` | Auto Loader 스트림의 **체크포인트 디렉터리**(STRING 리터럴) |
| `TABLE(table_name)` | `read_files`가 쓰는 **스트리밍 테이블 이름** (시점 지정 불가, **DBR 13.3 LTS+**) |

- 체크포인트의 소스 하위 경로(`.../sources/0`)를 지정하는 형태도 공식 예시에 있습니다.
- 권한: 스트리밍 테이블을 쓰면 DBR 17.1 이하는 **OWNER**, DBR 17.2 이상/Databricks SQL은 **SELECT와 MODIFY**가 필요하고, 외부 위치의 체크포인트를 직접 지정하면 해당 위치의 **READ FILES** 권한이 필요합니다.

## 4-2. 반환 컬럼

| 컬럼 | 타입 | 설명 | 조건 |
|---|---|---|---|
| `path` | STRING (PK) | 파일 경로 | — |
| `size` | BIGINT | 파일 크기(byte) | — |
| `create_time` | TIMESTAMP | 파일 생성 시각 | — |
| `discovery_time` | TIMESTAMP | 파일이 **발견된** 시각 | DBR 16.4+ |
| `processed_time` | TIMESTAMP | 파일이 **처리된** 시각. 배치가 실패해 재시도되면 **가장 최근 처리 시각** | DBR 16.4+ 스트림 + `cleanSource` 사용, **또는** DBR 18.2+ |
| `commit_time` | TIMESTAMP | 처리 후 체크포인트에 **커밋된** 시각. **미처리면 NULL** | 위와 같음 |
| `archive_time` | TIMESTAMP | 아카이브(이동/삭제)된 시각. 안 됐으면 NULL | DBR 16.4+ & `cleanSource` 사용 |
| `archive_mode` | STRING | `MOVE` / `DELETE` / (`OFF`면 NULL) | 위와 같음 |
| `move_location` | STRING | `MOVE`로 이동된 **전체 경로** | 위와 같음 |
| `source_id` | STRING | 소스 ID. 단일 위치면 `'0'` | — |
| `flow_name` | STRING | Lakeflow 파이프라인의 플로 이름. `table_name`을 안 주면 NULL | DBR 13.3+ |
| `ingestion_state` | STRING | 처리 상태 (아래) | `processed_time`과 같은 조건 |

> **중요**: `discovery_time`, `processed_time`, `commit_time`, `ingestion_state` 등은 **런타임 버전과 `cleanSource` 사용 여부에 따라 채워지지 않을 수 있습니다.** 값이 비어 있다고 파일이 처리되지 않았다는 뜻은 아닐 수 있으니, 사용 중인 DBR 버전을 먼저 확인하세요.
{: .notice--info}

## 4-3. `ingestion_state` 값

| 값 | 의미 |
|---|---|
| `NULL` | 아직 처리 안 됐거나, 상태를 **판단할 수 없음** |
| `PROCESSING` | 처리 중 |
| `INGESTED` | sink가 **최소 한 번** 처리함. `foreachBatch` 같은 **멱등하지 않은 sink**는 실패 시 **다시 처리될 수 있음** |
| `SKIPPED_CORRUPTED` | 파일이 **손상**되어 수집하지 않음 |
| `SKIPPED_MISSING` | 처리 중 파일을 **찾지 못해** 수집하지 않음 |
| `NOT_RECOGNIZED_BY_DBR` | 이후 버전에서 생긴 상태를 **이전 런타임이 인식하지 못할 때** 표시 (호환성용) |

**처리 완료의 공식 기준**: **`commit_time`이 NULL이 아니고 `ingestion_state`가 `INGESTED`인 파일만** 처리가 끝난 것입니다. 또 `commit_time`을 찍는 데 **보장된 지연 시간이 없어서**, 처리됐어도 한참 뒤에야 커밋 표시가 될 수 있습니다. 커밋이 표시되면 Auto Loader가 **그 파일을 다시 처리할 필요가 없다**는 뜻입니다.

## 4-4. 쓸 만한 조회 예시

```sql
-- ① 아직 커밋되지 않은 파일 (처리 중이거나 대기 중)
SELECT path, size, discovery_time, processed_time, ingestion_state
FROM cloud_files_state('/Volumes/main/landing/checkpoints/orders_ingest')
WHERE commit_time IS NULL
ORDER BY discovery_time;

-- ② 상태별 파일 수와 용량
SELECT ingestion_state, COUNT(*) AS files, SUM(size) AS bytes
FROM cloud_files_state('/Volumes/main/landing/checkpoints/orders_ingest')
GROUP BY ingestion_state;

-- ③ 처리가 확실히 끝난 파일 (공식 기준)
SELECT path, discovery_time, commit_time
FROM cloud_files_state('/Volumes/main/landing/checkpoints/orders_ingest')
WHERE commit_time IS NOT NULL AND ingestion_state = 'INGESTED';

-- ④ 수집하지 못하고 건너뛴 파일
SELECT path, ingestion_state
FROM cloud_files_state('/Volumes/main/landing/checkpoints/orders_ingest')
WHERE ingestion_state IN ('SKIPPED_CORRUPTED', 'SKIPPED_MISSING');

-- ⑤ 정리(cleanSource) 결과 확인
SELECT path, archive_mode, archive_time, move_location
FROM cloud_files_state('/Volumes/main/landing/checkpoints/orders_ingest')
WHERE archive_time IS NOT NULL;

-- ⑥ 스트리밍 테이블 이름으로 조회 (DBR 13.3 LTS+)
SELECT path, flow_name, discovery_time, ingestion_state
FROM cloud_files_state(TABLE(main.bronze.orders_raw_st));
```

```python
checkpoint_path = "/Volumes/main/landing/checkpoints/orders_ingest"
df_state = spark.sql(f"SELECT path, discovery_time, commit_time, ingestion_state "
                     f"FROM cloud_files_state('{checkpoint_path}')")
display(df_state)
```

- 파일을 **일정 기간 발견했는데 커밋이 안 된 목록**(①)을 주기적으로 점검하면 "누락 의심 파일" 감시가 됩니다.
- 스트림이 진행 중일 때 보는 **백로그 지표**(`numFilesOutstanding`, `numBytesOutstanding`)는 Streaming Query Listener로 별도로 확인할 수 있습니다.

# 5. 체크포인트 상태가 커질 때: `cloudFiles.maxFileAge`

파일 상태는 RocksDB에 계속 쌓입니다. **상태가 무한히 커지는 것을 막기 위해** 일정 기간이 지난 파일 이벤트를 만료시키는 옵션입니다.

| 항목 | 내용 (공식 문서) |
|---|---|
| 의미 | 파일 이벤트를 **중복 제거 목적**으로 추적하는 기간 |
| 기본값 | **없음(None)** |
| **최소값** | **14일** |
| 문서의 권고 | **수백만 파일/시간** 규모가 아니면 **조정하지 말 것**. 조정한다면 **90일처럼 보수적으로** |
| 성격 | 고용량 데이터셋의 **비용 제어 장치** |

## 위험: "중복"만이 아니라 "누락"도 생긴다

공식 문서는 너무 공격적으로 줄이면 **중복 적재 또는 파일 누락**이 생길 수 있다고 경고합니다.

- **오래 중단된 뒤 재시작**하면, `maxFileAge`보다 오래된 **파일 알림 이벤트는 무시**되고, 디렉터리 리스팅 모드에서도 **중단 중 생겼지만 `maxFileAge`보다 오래된 파일은 무시**됩니다. → **누락**
- 이미 처리한 파일의 기록이 만료된 뒤 그 파일이 다시 보이면 **다시 처리**됩니다. → **중복**
- 디렉터리 리스팅 모드에서 `maxFileAge`를 `1 month`에서 `2 months`로 **늘려** 재시작하면, **1개월보다 오래되고 2개월보다 최근인 파일이 재처리**됩니다.
- **스트림을 처음 시작할 때** 이 옵션을 설정하면 **`maxFileAge`보다 오래된 데이터는 수집되지 않습니다.** 오래된 데이터를 적재하려면 **처음에는 설정하지 말고, 이후 실행부터 설정**하세요.
- 만료된 이벤트는 RocksDB에서 **삭제 표시(tombstone)** 로 남아서, 저장소 사용량이 **일시적으로 늘었다가 안정**될 수 있습니다.

# 6. 원본 파일 정리: `cloudFiles.cleanSource`

처리가 끝난 파일을 **이동(MOVE)하거나 삭제(DELETE)** 해서 저장 비용을 줄이고 **이후 리스팅 시간을 줄이는** 기능입니다. **DBR 16.4 이상**에서 사용합니다.

## 6-1. 옵션

| 옵션 | 기본값 | 설명 |
|---|---|---|
| `cloudFiles.cleanSource` | `OFF` | `OFF`: 아무것도 안 함 / `MOVE`: 보존 기간 후 이동 / `DELETE`: 보존 기간 후 삭제 |
| `cloudFiles.cleanSource.retentionDuration` | `30 days` | 처리된 파일이 정리 **후보가 되기까지 기다리는 시간**. **DELETE는 7일 초과** 필요, **MOVE는 최소 제한 없음** |
| `cloudFiles.cleanSource.moveDestination` | 없음 | `MOVE` 때 이동할 경로 (필수) |
| `cloudFiles.cleanSource.waitForCompletion` | `false` | **DBR 19+**. 정리가 끝날 때까지 스트림을 유지 |

## 6-2. MOVE 목적지의 조건 (자주 틀리는 부분)

- 소스 디렉터리의 **하위 경로가 아니어야** 합니다. **하위에 두면 이동된 파일이 다시 수집됩니다.**
- 소스와 **같은 external location, 볼륨, DBFS 마운트** 안에 있어야 합니다. **다른 버킷/컨테이너로의 이동은 지원되지 않으며 오류**가 납니다.
- Auto Loader에게 해당 디렉터리의 **쓰기 권한**이 있어야 합니다.

## 6-3. 언제 정리가 일어나나 (타이밍)

```
N번째 실행   파일 수집(처리)
N+1번째 실행 그 파일의 commit_time 이 기록됨
N+2번째 실행부터 정리 "후보"가 될 수 있음
             + commit_time 부터 retentionDuration 이 지나야 실제 MOVE/DELETE
```

- **`commit_time`은 파일이 수집된 다음 실행(N+1)에서 기록됩니다.** 정리 후보가 될 수 있는 가장 이른 시점은 **N+2번째 실행**이며, 보존 기간은 **`commit_time`부터** 계산합니다. 두 조건을 **모두** 만족해야 합니다.
- 정리는 **실행할 새 파일 배치가 있을 때만** 시작됩니다. 백그라운드 프로세스가 아닙니다. 새 파일이 오지 않으면 보존 기간이 지난 파일도 **다음에 새 배치가 처리될 때까지 정리되지 않습니다.** (`waitForCompletion`을 켜도 마찬가지)
- 정리는 기본적으로 **최선 노력(best-effort)** 입니다. 스트림이 먼저 끝나면 정리가 중단되고, 못 한 파일은 **다음 실행**에서 처리됩니다. 보존 기간이 끝났다고 **즉시** 정리된다는 보장은 없습니다.
- 정리 시점의 **현재 설정**(MOVE/DELETE)이 적용됩니다. 처음엔 MOVE였다가 30일 뒤 DELETE로 바꾸면 **삭제**됩니다.

## 6-4. 사용 전 고려사항

- **같은 원본 폴더를 여러 스트림이 읽으면 사용하지 마세요.** 가장 빠른 스트림이 파일을 정리해서 느린 스트림은 **그 파일을 읽지 못합니다.**
- 켜면 체크포인트에 **추가 상태**를 유지해서 **성능 오버헤드**가 생기지만, 대신 `cloud_files_state`로 **관측성**이 좋아집니다.
- `DELETE`를 쓸 때는 **버킷 버전 관리(versioning)를 켜서 삭제가 소프트 삭제**가 되게 하고, **수명 주기 정책으로 오래된 삭제 버전을 60~90일 뒤 정리**하라고 권장합니다. (설정 실수 대비)
- `MOVE`로 보관할 때는 **아카이브 경로에 수명 주기 정책**을 걸어 더 저렴한 스토리지 계층(Glacier, Cool/Archive, Coldline 등)으로 넘길 수 있습니다.
- 감사·규정 준수로 원본 보관이 필요하면 `MOVE`, 비용 절감이 목적이면 `DELETE`가 문서의 안내입니다.

> **같은 이름의 다른 옵션 주의**: Delta/일반 파일 스트리밍 소스에는 `cleanSource`(`off/delete/archive`)와 `sourceArchiveDir` 옵션이 **별도로** 있습니다. 이 글의 `cloudFiles.cleanSource`(`OFF/DELETE/MOVE`, `cloudFiles.` 접두사)와 **다른 옵션**이며 Delta 테이블 스트리밍에는 적용되지 않습니다.
{: .notice--info}

# 7. 코드: 변경 전/후

## 7-1. 변경 전 (오류가 나거나 의도와 다르게 동작하는 예)

```python
df = (spark.readStream.format("cloudFiles")
      .option("cloudFiles.format", "parquet")
      .option("cloudFiles.schemaLocation", "s3://landing-bucket/_schema/orders")
      .option("cloudFiles.cleanSource", "MOVE")
      .option("cloudFiles.cleanSource.retentionDuration", "7 days")
      .option("cloudFiles.cleanSource.moveDestination", "s3://archive-bucket/orders/")   # ← 다른 버킷
      .load("s3://landing-bucket/orders/"))
```

문제점
1. 아카이브 경로가 **다른 버킷** → 공식 문서상 **지원되지 않고 오류**.
2. 같은 버킷이라도 아카이브가 `orders/` **하위**에 있으면 **다시 수집**됩니다.
3. "처리 끝나고 7일 뒤 이동"이라고 이해하기 쉽지만 실제로는 **`commit_time`(다음 실행에서 기록) 기준**이고 **최선 노력**입니다.

## 7-2. 변경 후

```python
landing  = "s3://example-bucket/landing/orders/"          # 원본
archive  = "s3://example-bucket/archive/orders/"          # 같은 버킷, 원본의 하위가 아닌 형제 경로
ckpt     = "/Volumes/main/landing/checkpoints/orders_ingest"

df = (spark.readStream.format("cloudFiles")
      .option("cloudFiles.format", "parquet")
      .option("cloudFiles.schemaLocation", ckpt)
      .option("cloudFiles.cleanSource", "MOVE")
      .option("cloudFiles.cleanSource.moveDestination", archive)
      .option("cloudFiles.cleanSource.retentionDuration", "14 days")
      # .option("cloudFiles.maxFileAge", "90 days")   # 수백만 파일/시간 규모가 아니면 설정하지 않는 것이 문서 권고
      .load(landing))

(df.writeStream
   .option("checkpointLocation", ckpt)
   .trigger(availableNow=True)
   .toTable("main.bronze.orders_raw"))
```

- 아카이브는 **같은 버킷의 원본과 겹치지 않는 경로**로, 쓰기 권한을 확인합니다.
- 아카이브 경로에 클라우드 **수명 주기 정책**을 걸어 저비용 계층으로 보냅니다. (Databricks 밖에서 설정)
- 정리 상태는 4장의 ⑤번 쿼리로 확인합니다.
- 체크포인트에는 **수명 주기 정책을 걸지 않습니다.**

# 8. 흔한 설명과 공식 문서의 차이

| 자주 보이는 설명 | 공식 문서 |
|---|---|
| MOVE 목적지는 **같은 버킷을 권장**, 다른 버킷은 비용이 든다 | **다른 버킷/컨테이너는 지원되지 않고 오류**. 같은 external location/볼륨/마운트 **필수** |
| DELETE는 **최소 7일 이상** | "**greater than 7 days**" (7일 **초과**). MOVE는 최소 제한 없음 |
| `commit_time`이 찍힌 뒤 N일 후 정리 | 맞지만, `commit_time`은 **다음 실행(N+1)에서 기록**되고 **새 파일 배치가 있어야** 정리가 시작됨. **최선 노력**이며 즉시 보장 없음 |
| `commit_time`이 NULL이 아니면 처리 완료 | **`commit_time` NOT NULL + `INGESTED`** 가 완료. 또한 이 컬럼들은 **런타임/옵션 조건**에서만 채워짐 |
| `maxFileAge`를 너무 줄이면 **중복**이 생긴다 | **중복과 누락 모두** 가능. **최소 14일**, 처음 시작 때 설정하면 **오래된 데이터 미수집** |
| MOVE 목적지에 대한 제한은 없다 | 소스의 **하위 경로면 다시 수집**됨, **쓰기 권한 필요** |
| `ingestion_state`는 INGESTED/PROCESSING/SKIPPED_CORRUPTED 등 | `SKIPPED_MISSING`, `NOT_RECOGNIZED_BY_DBR`도 있음. `INGESTED`는 **최소 한 번** 처리(멱등하지 않은 sink는 재처리 가능) |
| `cleanSource`가 모든 스트림에 적용 | Auto Loader는 `cloudFiles.cleanSource`, Delta/파일 소스는 **별도 옵션**(`cleanSource`+`sourceArchiveDir`) |

# 9. 상황별 선택

| 상황 | 선택 |
|---|---|
| 처리 파일을 **감사용으로 보관**해야 함 | `MOVE` + 아카이브 경로 수명 주기 정책 |
| **저장 비용/리스팅 시간**을 줄이고 싶음 | `DELETE` + **버킷 버전 관리** + 삭제 버전 정리 정책 |
| 같은 원본을 **여러 스트림이 소비** | `cleanSource` **사용 금지**. 스트림별 원본 분리를 설계 |
| 체크포인트 상태가 커서 **시작이 느림** | **DBR 15.4 LTS+** 로 업그레이드, 파일 이벤트 기반 **file notification 모드**(문서 권장) 검토, `maxFileAge`는 **마지막 수단**(90일 등 보수적으로) |
| 오래 **중단 후 재시작**할 일이 있음 | `maxFileAge`로 인한 **누락** 가능성을 먼저 점검 |
| **오래된 파일까지 처음 적재** | 첫 실행에는 `maxFileAge`를 **설정하지 않음** |
| 정리 대상이 **오래 쌓여 있음** | 새 파일이 없는 실행은 정리를 시작하지 않으므로 **새 배치가 있는 실행**이 필요 |

# 10. 체크리스트

- [ ] 사용 중인 **DBR 버전**을 확인했는가? (`cloud_files_state` 일부 컬럼은 16.4+/18.2+)
- [ ] 처리 완료를 **`commit_time IS NOT NULL AND ingestion_state='INGESTED'`** 로 판단하는가?
- [ ] 체크포인트 위치에 **수명 주기 정책이 없는가**? 위치를 **바꾸지 않았는가**?
- [ ] `maxFileAge`는 꼭 필요한가? 필요하다면 **14일 이상, 가급적 90일처럼 보수적**인가? 장기 중단 시 **누락** 위험을 점검했는가?
- [ ] MOVE 목적지가 **같은 버킷/볼륨**이고 **소스의 하위가 아닌가**? **쓰기 권한**이 있는가?
- [ ] DELETE라면 **버킷 버전 관리**와 **보존 기간 7일 초과**를 확인했는가?
- [ ] **여러 스트림이 같은 원본**을 읽지 않는가?
- [ ] `foreachBatch` 같은 **멱등하지 않은 sink**라면 재처리를 견디는가? (→ 멱등성 글)

# 11. 한 줄 정리

- Auto Loader는 **체크포인트의 RocksDB**로 처리한 파일을 기억해 exactly-once를 보장하고, `cloud_files_state`로 그 상태를 조회한다. **완료 = `commit_time` NOT NULL + `INGESTED`**.
- 상태가 커지는 문제는 **DBR 15.4+, 파일 이벤트 모드**를 먼저 보고, `maxFileAge`는 **최소 14일·권장 90일·중복과 누락 위험**을 알고 쓴다.
- 원본 정리는 `cloudFiles.cleanSource`로 한다. **MOVE는 같은 버킷/볼륨 안의 형제 경로**, **DELETE는 7일 초과 + 버전 관리**, 타이밍은 **다음 실행의 `commit_time` 기준 + 최선 노력**이다.
- 요약·블로그 설명은 **공식 문서와 대조**해서 쓴다. 특히 MOVE 목적지, 보존 기간 최소값, 정리 타이밍이 자주 틀린다.

# 관련 용어 글
Auto Loader · Auto Loader 파일 추적과 체크포인트 · cloud_files_state · cloudFiles.maxFileAge · cloudFiles.cleanSource · 멱등성

## 🔗 참고
- [cloud_files_state table-valued function](https://docs.databricks.com/aws/en/sql/language-manual/functions/cloud_files_state)
- [Auto Loader options](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/options)
- [Clean up processed files with Auto Loader](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/clean-source)
- [Configure Auto Loader for production workloads](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/production)
- [What is Auto Loader?](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/)
- [Auto Loader file detection modes](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/file-detection-modes)
- [Auto Loader FAQ](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/faq)
