---
title:  "[DE/용어] cloud_files_state (테이블 반환 함수)"
excerpt: "Auto Loader가 체크포인트에 저장한 파일별 상태를 SQL로 조회하는 함수"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, Auto Loader, cloud_files_state, Monitoring]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[cloud_files_state table-valued function](https://docs.databricks.com/aws/en/sql/language-manual/functions/cloud_files_state)    
[Configure Auto Loader for production workloads](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/production)
{: .notice--warning}

# 0. 이 글을 읽기 전에

**Auto Loader 파일 추적과 체크포인트** 글에서 "파일별 상태가 체크포인트에 저장된다"고 했습니다. 이 글은 그 상태를 **눈으로 확인하는 방법**입니다.

# 1. 한 줄 정의

`cloud_files_state`는 **Auto Loader 또는 `read_files` 스트림의 파일 단위 상태를 테이블 형태로 반환**하는 **테이블 반환 함수(table-valued function)** 입니다. **Databricks Runtime 11.3 LTS 이상**, Databricks SQL에서 사용할 수 있습니다.

# 2. 문법

```sql
cloud_files_state( { TABLE ( table_name ) | checkpoint } )
```

```sql
-- 체크포인트 경로로
SELECT * FROM cloud_files_state('/Volumes/main/landing/_checkpoints/orders');

-- 소스 하위 경로로 (공식 예시 형태)
SELECT path FROM cloud_files_state('/some/checkpoint/sources/0');

-- 스트리밍 테이블 이름으로 (DBR 13.3 LTS+)
SELECT path FROM cloud_files_state(TABLE(main.bronze.orders_raw_st));
```

- `TABLE(table_name)`의 테이블은 **`read_files`가 쓰는 스트리밍 테이블**이어야 하고, **시점(temporal) 지정은 불가**합니다.

# 3. 반환 컬럼

| 컬럼 | 설명 |
|---|---|
| `path` | 파일 경로 (기본키) |
| `size` | 파일 크기(byte) |
| `create_time` | 파일 생성 시각 |
| `discovery_time` | 파일 **발견** 시각 (DBR 16.4+) |
| `processed_time` | 파일 **처리** 시각. 배치가 실패해 재시도되면 **가장 최근 처리 시각** (조건부, 아래) |
| `commit_time` | 처리 후 체크포인트에 **커밋**된 시각. **미처리면 NULL** (조건부) |
| `archive_time` / `archive_mode` / `move_location` | `cleanSource`로 **정리(이동/삭제)** 된 시각, 방식, 이동 경로 |
| `source_id` | 소스 ID (단일 위치면 `'0'`) |
| `flow_name` | Lakeflow 파이프라인의 플로 이름 (`table_name`을 안 주면 NULL, DBR 13.3+) |
| `ingestion_state` | 처리 상태 (아래, 조건부) |

> **조건부 컬럼**: `processed_time`, `commit_time`, `ingestion_state`는 **DBR 16.4+ 스트림에서 `cloudFiles.cleanSource`를 켠 경우, 또는 DBR 18.2 이상**에서 채워집니다. `archive_*`, `move_location`은 DBR 16.4+에서 `cleanSource`를 쓸 때 의미가 있습니다. **값이 비어 있다고 처리 안 된 것이 아닐 수 있습니다.**
{: .notice--info}

## `ingestion_state`

| 값 | 의미 |
|---|---|
| `NULL` | 아직 처리 안 됨, 또는 상태를 **판단할 수 없음** |
| `PROCESSING` | 처리 중 |
| `INGESTED` | sink가 **최소 한 번** 처리. 멱등하지 않은 sink(예: `foreachBatch`)는 실패 시 **재처리될 수 있음** |
| `SKIPPED_CORRUPTED` | **손상**된 파일이라 수집 안 함 |
| `SKIPPED_MISSING` | 처리 중 **파일을 찾지 못해** 수집 안 함 |
| `NOT_RECOGNIZED_BY_DBR` | 이후 버전의 상태를 **이전 런타임이 인식하지 못함** (호환성용) |

# 4. "처리 완료"를 판단하는 법

공식 문서: **`commit_time`이 NULL이 아니고, `ingestion_state`가 `INGESTED`인 파일만** 처리가 끝난 것입니다.

- `commit_time`을 찍는 데는 **보장된 지연 시간이 없습니다.** 처리됐어도 **한참 뒤에야** 커밋 표시가 될 수 있습니다.
- 커밋 표시가 됐다는 것은 Auto Loader가 **그 파일을 다시 처리할 필요가 없다**는 뜻입니다.

# 5. 활용 예

```sql
-- 발견됐지만 아직 커밋 안 된 파일
SELECT path, discovery_time, ingestion_state
FROM cloud_files_state('/Volumes/main/landing/_checkpoints/orders')
WHERE commit_time IS NULL;

-- 상태별 건수
SELECT ingestion_state, COUNT(*) FROM cloud_files_state('/Volumes/main/landing/_checkpoints/orders')
GROUP BY ingestion_state;

-- 건너뛴 파일 점검
SELECT path FROM cloud_files_state('/Volumes/main/landing/_checkpoints/orders')
WHERE ingestion_state IN ('SKIPPED_CORRUPTED', 'SKIPPED_MISSING');
```

# 6. 권한과 제한

- **스트리밍 테이블 이름**을 쓸 때: DBR 17.1 이하는 **OWNER**, DBR 17.2 이상/Databricks SQL은 **SELECT와 MODIFY**
- **외부 위치의 체크포인트**를 직접 지정할 때: 그 위치의 **READ FILES** 권한
- `cloud_files_state`를 참조하는 **뷰**를 읽으려면 뷰의 SELECT와 스트리밍 테이블에 필요한 권한을 **모두** 가져야 합니다.

# 7. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "`commit_time`이 비어 있으면 처리 안 된 파일" | **런타임/옵션 조건**에서만 채워짐. 먼저 DBR 버전 확인 |
| "`INGESTED`면 항상 한 번만 처리됨" | **최소 한 번**. 멱등하지 않은 sink는 재처리 가능 |
| "함수로 체크포인트 상태를 바꿀 수 있다" | **조회 전용** |

# 8. 한 줄 정리

- `cloud_files_state` = **Auto Loader 체크포인트의 파일 단위 상태를 조회하는 SQL 함수** (DBR 11.3 LTS+).
- 완료 판정은 **`commit_time` NOT NULL + `INGESTED`**. 일부 컬럼은 **DBR 16.4+/18.2+ 조건**에서만 채워진다.
- 누락 의심 파일은 **`commit_time IS NULL`** 과 **`SKIPPED_*`** 로 찾는다.

# 관련 용어
Auto Loader, Auto Loader 파일 추적과 체크포인트, cloudFiles.cleanSource

## 🔗 참고
- [cloud_files_state table-valued function](https://docs.databricks.com/aws/en/sql/language-manual/functions/cloud_files_state)
- [Configure Auto Loader for production workloads](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/production)
