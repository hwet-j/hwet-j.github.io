---
title:  "[DE/용어] cloudFiles.cleanSource"
excerpt: "처리가 끝난 원본 파일을 자동으로 이동(MOVE)하거나 삭제(DELETE)하는 옵션과 타이밍·제약"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, Auto Loader, cleanSource, Source Cleanup]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[Clean up processed files with Auto Loader](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/clean-source)    
[Auto Loader options](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/options)    
[Configure Auto Loader for production workloads](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/production)
{: .notice--warning}

# 0. 이 글을 읽기 전에

Auto Loader는 파일을 읽어도 **원본을 그대로 둡니다.** 시간이 지나면 랜딩 폴더에 파일이 계속 쌓이고, 리스팅이 느려지고 저장 비용이 늘어납니다. 이 글은 처리된 파일을 **자동으로 정리**하는 `cloudFiles.cleanSource`를 다룹니다. **DBR 16.4 이상**에서 사용합니다.

# 1. 한 줄 정의

`cloudFiles.cleanSource`는 Auto Loader가 **처리가 끝난 원본 파일을 보존 기간 뒤에 이동(MOVE)하거나 삭제(DELETE)** 하게 하는 옵션입니다. 정리하면 **저장 비용이 줄고 이후 리스팅 시간도 짧아집니다.**

# 2. 모드

| 값 | 동작 |
|---|---|
| `OFF` (기본) | 파일을 **이동/삭제하지 않음** |
| `MOVE` | 보존 기간 후 **`cloudFiles.cleanSource.moveDestination`으로 이동** |
| `DELETE` | 보존 기간 후 **삭제** |

# 3. 관련 옵션

| 옵션 | 기본값 | 설명 |
|---|---|---|
| `cloudFiles.cleanSource.retentionDuration` | `30 days` | 처리된 파일이 **정리 후보가 되기까지** 기다리는 시간. 형식: `14 days`, `2 weeks`, `1 month` 등. **DELETE는 7일 초과**여야 하고 **MOVE는 최소 제한 없음** |
| `cloudFiles.cleanSource.moveDestination` | 없음 | `MOVE`일 때 이동할 경로(**필수**) |
| `cloudFiles.cleanSource.waitForCompletion` | `false` | **DBR 19+**. 정리가 끝날 때까지 스트림을 유지 (기본은 정리가 끝나기 전에 스트림이 끝나면 **정리가 중단**됨) |

# 4. MOVE 목적지 조건

- 소스 디렉터리의 **하위 경로가 아닐 것** — 하위에 두면 **이동된 파일이 다시 수집**됩니다.
- 소스와 **같은 external location, 볼륨 또는 DBFS 마운트** 안일 것 — **다른 버킷/컨테이너로의 이동은 지원되지 않으며 오류**.
- Auto Loader에 **쓰기 권한**이 있을 것.

```
✅ 원본  s3://example-bucket/landing/orders/
   아카이브 s3://example-bucket/archive/orders/        (같은 버킷, 형제 경로)

❌ 아카이브 s3://example-bucket/landing/orders/archive/   (원본의 하위 → 다시 수집됨)
❌ 아카이브 s3://other-bucket/archive/orders/              (다른 버킷 → 오류)
```

# 5. 언제 정리되나: 타이밍을 정확히

```
N번째 실행    파일 수집(처리)
N+1번째 실행  그 파일의 commit_time 이 기록됨
N+2번째 실행  이후부터 정리 "후보" 가능 (commit_time 부터 retentionDuration 경과 필요)
```

- `commit_time`은 **수집된 다음 실행(N+1)에서 기록**되고, 후보가 될 수 있는 가장 이른 시점은 **N+2번째 실행**입니다.
- 보존 기간은 **`commit_time`부터** 셉니다. 둘을 **모두** 만족해야 이동·삭제됩니다.
- 정리는 **처리할 새 파일 배치가 있을 때만** 시작합니다. **백그라운드로 상시 도는 작업이 아닙니다.** 새 파일이 안 오면 보존 기간이 지난 파일도 **다음 배치 처리 때까지 정리되지 않습니다.**
- 정리는 **최선 노력(best-effort)** 입니다. 스트림 처리가 끝나면 정리가 **중단**되고 못 한 파일은 **다음 실행**에서 처리됩니다. 보존 기간이 끝나자마자 정리된다는 **보장이 없습니다.**
- 정리 시점의 **현재 설정**이 적용됩니다. (예: MOVE로 처리했는데 30일 뒤 DELETE로 바꿨다면 **삭제**)

# 6. 사용 전 고려사항

| 항목 | 내용 |
|---|---|
| **여러 스트림이 같은 원본** | **사용하지 않기.** 가장 빠른 스트림이 정리해서 느린 스트림이 **그 파일을 읽지 못함** |
| 체크포인트 **오버헤드** | 추가 상태를 유지해서 **성능 오버헤드**가 있음. 대신 `cloud_files_state`로 **관측성 향상** |
| **DELETE의 안전장치** | **버킷 버전 관리(versioning)** 를 켜서 소프트 삭제로 만들고, **수명 주기 정책으로 오래된 삭제 버전을 60~90일 후 정리** 권장 |
| **MOVE로 보관** | 아카이브 경로에 **수명 주기 정책**을 걸어 **저비용 계층**(Glacier, Cool/Archive, Coldline 등)으로 이동 |
| 용도 | **감사·규정 준수**로 보관 → `MOVE`, **비용 절감** → `DELETE` |

# 7. 설정 예

```python
df = (spark.readStream.format("cloudFiles")
      .option("cloudFiles.format", "json")
      .option("cloudFiles.cleanSource", "MOVE")
      .option("cloudFiles.cleanSource.moveDestination", "s3://example-bucket/archive/orders/")
      .option("cloudFiles.cleanSource.retentionDuration", "14 days")
      .option("cloudFiles.schemaLocation", "/Volumes/main/landing/_checkpoints/orders")
      .load("s3://example-bucket/landing/orders/"))
```

정리 결과는 `cloud_files_state`의 `archive_time`, `archive_mode`, `move_location` 컬럼으로 확인합니다. (→ cloud_files_state 글)

# 8. 이름이 같은 다른 옵션

Delta·일반 파일 스트리밍 소스의 **`cleanSource`**(`off/delete/archive`) 와 **`sourceArchiveDir`** 옵션은 **별개**입니다. 이 글의 `cloudFiles.cleanSource`(`OFF/DELETE/MOVE`)와 이름이 비슷하지만 **다른 옵션**이고, Delta 테이블 스트리밍에는 적용되지 않습니다.

# 9. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "처리 끝나면 곧바로 정리된다" | **N+2번째 실행부터 후보**, **`commit_time` 기준 보존 기간**, **최선 노력** |
| "새 파일이 없어도 백그라운드로 정리된다" | **새 파일 배치가 있어야** 시작 |
| "MOVE는 다른 버킷으로도 된다" | **같은 위치 안에서만**. 다른 버킷은 **오류** |
| "DELETE도 1일 뒤로 설정 가능" | **7일 초과**여야 함 |
| "아카이브를 원본 폴더 안에 두면 편하다" | **다시 수집**됨 |

# 10. 한 줄 정리

- `cleanSource` = **처리된 원본 파일을 보존 기간 후 MOVE/DELETE**하는 옵션 (DBR 16.4+, 기본 `OFF`).
- **MOVE는 같은 위치 안의 형제 경로**, **DELETE는 7일 초과 + 버킷 버전 관리 권장**, **여러 스트림이 같은 원본을 읽으면 금지**.
- 정리는 **다음 실행의 `commit_time` 이후 + 새 배치가 있을 때 + 최선 노력**으로 일어난다.

# 관련 용어
Auto Loader, Auto Loader 파일 추적과 체크포인트, cloud_files_state, cloudFiles.maxFileAge

## 🔗 참고
- [Clean up processed files with Auto Loader](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/clean-source)
- [Auto Loader options](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/options)
- [Configure Auto Loader for production workloads](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/production)
