---
title:  "[DE/Databricks] Databricks 캐시 총정리: Disk Cache, Result Cache, UI Cache 그리고 대시보드 캐시"
excerpt: "어떤 캐시가 무엇을 어디에 저장하고 언제 사라지는지, 흔한 설명과 공식 문서의 차이까지 한 번에 정리"

categories:
  - DataEngineering
tags:
  - [Databricks, Cache, Disk Cache, Result Cache, Spark Cache, Dashboard]

toc: true
toc_sticky: true
published: true

date: 2026-05-06
last_modified_at: 2026-11-20
---

업무 중 정리한 내용을 일반화해서 정리했습니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서(AWS 기준)를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[Optimize performance with caching](https://docs.databricks.com/aws/en/optimizations/disk-cache) ·
[Query caching](https://docs.databricks.com/aws/en/sql/user/queries/query-caching) ·
[Dataset optimization and caching](https://docs.databricks.com/aws/en/dashboards/caching) ·
[USE_CACHED_RESULT](https://docs.databricks.com/aws/en/sql/language-manual/parameters/use_cached_result) ·
[CACHE TABLE](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-aux-cache-cache-table) ·
[Serverless compute limitations](https://docs.databricks.com/aws/en/compute/serverless/limitations) ·
[성능 효율 모범 사례](https://docs.databricks.com/aws/en/lakehouse-architecture/performance-efficiency/best-practices)
{: .notice--warning}

# 1. 상황

Databricks를 쓰다 보면 이런 일이 생깁니다.

- 같은 쿼리인데 **어제는 빨랐는데 클러스터를 재시작했더니 느리다.**
- 데이터를 갱신했는데 **대시보드에는 예전 숫자가 그대로** 보인다.
- 쿼리에 `current_timestamp()`를 넣었더니 **캐시가 안 먹는다.**
- 벤치마크를 하는데 **두 번째 실행부터 비정상적으로 빠르다.**

원인은 대부분 **캐시가 여러 층으로 있고, 층마다 저장 대상·위치·수명·무효화 방식이 다르기** 때문입니다. 이 글은 그 지도를 공식 문서 기준으로 정리하고, 흔히 쓰이는 설명에서 **공식 문서와 다른 부분**을 바로잡습니다.

# 2. 캐시 지도 한눈에 보기

| 캐시 | 무엇을 저장하나 | 어디에 | 적용 대상 | 수명 | 무효화 |
|---|---|---|---|---|---|
| **Disk cache** | **Parquet 데이터 파일의 복사본** (Delta 포함) | 워커 노드 **로컬 디스크(SSD)** | 클러스터, SQL 웨어하우스 | 클러스터와 동일(재시작 시 비워짐), LRU로 밀려남 | 파일이 생성·삭제·수정·덮어쓰기되면 **자동** |
| **Spark cache** (`CACHE TABLE`, `.cache()`) | DataFrame/쿼리 결과의 **메모리 블록**(저장 수준에 따라) | 노드 메모리/디스크 | 클러스터 (서버리스는 제한) | 수동(`unpersist`) 또는 LRU, 워커가 사라지면 소실 | 수동 |
| **Local result cache** | **쿼리 최종 결과** | SQL 웨어하우스 클러스터의 **메모리** | **SQL 웨어하우스** | 클러스터 수명 또는 캐시가 찰 때까지, **최대 24시간** | 기저 테이블이 갱신되면 **무효화** |
| **Remote result cache** | 쿼리 최종 결과 | **워크스페이스 시스템 데이터**로 영속화 | **서버리스 SQL 웨어하우스 전용** | **24시간**, 웨어하우스 재시작에도 유지 | 기저 테이블이 갱신되면 **무효화** |
| **Databricks SQL UI cache** | 쿼리 결과와 SQL 편집기 시각화 | 계정의 Databricks 파일시스템 | SQL 편집기, **레거시 SQL 대시보드** | **최대 7일**, 사용자별 | 기저 테이블 갱신, 쿼리 재실행 |
| **AI/BI 대시보드 캐시** | 대시보드 데이터셋 쿼리 결과 | (문서에 위치 명시 없음) | **AI/BI 대시보드** | **24시간**(최선 노력) | **데이터가 바뀌어도 자동 무효화되지 않음**, 스케줄로 갱신 |

> UI cache는 **AI/BI 대시보드에 적용되지 않습니다.** AI/BI 대시보드는 위 표의 마지막 행처럼 **별도의 캐시 동작**을 가집니다. 이 둘을 섞어서 "대시보드 캐시는 7일"이라고 알고 있으면 틀립니다.
{: .notice--info}

# 3. 캐시가 겹치는 순서 (개념도)

공식 문서 내용을 종합한 개념도입니다. 상세한 내부 순서는 제품 버전에 따라 다를 수 있습니다.

```
[사용자가 대시보드/쿼리를 연다]
   │
   ├─ AI/BI 대시보드  → ① 대시보드 캐시(24h)  ── 없으면 ──┐
   ├─ SQL 편집기/레거시 대시보드 → UI cache(≤7d)          │
   │                                                       ▼
   │                                         ② 쿼리 결과 캐시 (SQL 웨어하우스)
   │                                              local(메모리) → remote(서버리스) → 없으면 실행
   │                                                       │
   │                                                       ▼ (실행)
   │                                         ③ Disk cache (로컬 SSD의 Parquet 복사본)
   │                                                       │ 없으면
   │                                                       ▼
   └─────────────────────────────────────────▶ ④ 원격 클라우드 스토리지 (S3 등)
```

- 위층 캐시가 **적중하면 아래층은 아예 건드리지 않습니다.** 대시보드 캐시나 remote result cache 적중은 SQL 웨어하우스를 **시작하지 않을 수도** 있습니다. (대시보드 캐시 문서: "캐시에서 결과를 제공해도 웨어하우스가 시작되지 않는다")
- 맨 아래 **Disk cache**는 쿼리가 **실제로 실행될 때** 원격 스토리지 읽기를 줄이는 역할입니다.

# 4. Disk cache

## 4-1. 무엇이고 어떻게 동작하나

- 노드가 원격 스토리지에서 **Parquet 데이터 파일을 가져올 때, 그 사본을 자동으로 로컬 저장소에 빠른 중간 포맷으로 저장**합니다. 이후 같은 데이터를 읽으면 **로컬에서 읽어 훨씬 빠릅니다.**
- **모든 Parquet 데이터 파일**에 적용되며 Delta Lake 테이블도 포함됩니다.
- **첫 읽기 때** 저장되고(Cold start), 활성화되어 있어야 합니다. **반복·반복적(iterative) 쿼리**에서 효과가 큽니다.
- 쿼리 **조건이 달라도 같은 데이터 파일을 읽는다면** 캐시된 파일을 재사용합니다. (캐시 단위가 **파일**이기 때문. 쿼리 결과 자체를 저장하는 것이 아님)
- 서브쿼리의 **임의의 중간 결과는 저장하지 못합니다.** (그건 Spark cache의 영역)
- 과거 이름은 **Delta cache, DBIO cache**였고, Delta Lake 프로토콜의 일부로 오해되지 않도록 **disk cache로 이름을 바꿨습니다.** 이 동작은 Databricks 전용 기능입니다.

## 4-2. 일관성: 데이터가 바뀌면?

- Disk cache는 **데이터 파일이 생성·삭제·수정·덮어쓰기되는 것을 자동으로 감지**해서 내용을 갱신합니다. 테이블 데이터를 쓰거나 수정·삭제해도 **명시적으로 캐시를 무효화할 필요가 없고**, 오래된 항목은 **자동으로 무효화·제거**됩니다.
- 즉 INSERT/UPDATE/DELETE로 **바뀐 파일의 캐시만** 무효화되고 바뀌지 않은 파일의 캐시는 그대로 유지됩니다.

## 4-3. 용량 관리 (공식 문서)

- 로컬 SSD가 있는 워커의 **최대 절반 공간**까지 사용하도록 구성됩니다.
- 공간이 부족하면 **LRU(가장 오래 안 쓴 것부터)** 로 제거됩니다. 클러스터를 **재시작하면 수동으로 비워집니다.**
- 직접 조정하려면 클러스터 생성 시 Spark 설정을 지정합니다.

```ini
spark.databricks.io.cache.maxDiskUsage 50g          # 노드당 캐시 데이터용 디스크 (byte 단위 값도 가능)
spark.databricks.io.cache.maxMetaDataCache 1g       # 노드당 캐시 메타데이터용 디스크
spark.databricks.io.cache.compression.enabled false # 캐시 데이터를 압축 저장할지
```

## 4-4. 켜고 끄고 확인하기

```python
spark.conf.get("spark.databricks.io.cache.enabled")            # 현재 설정 확인
spark.conf.set("spark.databricks.io.cache.enabled", "false")   # 끄기 (또는 "true")
```

- **끄면** 이미 로컬에 있는 데이터가 삭제되는 것이 아니라, **쿼리가 캐시에 새 데이터를 넣거나 캐시에서 읽는 것이 막힙니다.**
- 가장 쉬운 사용법은 **SSD가 달린 캐시 가속(cache-accelerated) 워커 타입**을 고르는 것입니다. 이런 인스턴스는 **자동으로 최적 설정**됩니다. 지원하는 워커에서는 기본으로 켜져 있습니다.
- **SQL 웨어하우스와 DBR 14.2 이상**에서는 `CACHE SELECT` 명령이 **무시**되고 **개선된 디스크 캐시 알고리즘**이 대신 쓰입니다.

# 5. Spark cache (`CACHE TABLE`, `df.cache()`)와의 차이

| 항목 | Disk cache | Spark cache |
|---|---|---|
| 저장 형태 | 워커의 **로컬 파일** | **메모리 블록** (저장 수준에 따라 다름) |
| 대상 | S3, ABFS 등의 **Parquet 테이블** | **모든 DataFrame/RDD** (CSV, JSON, ORC, 임의의 서브쿼리 결과 포함) |
| 시작 | **자동**, 첫 읽기 시 | **수동**, 코드 변경 필요 |
| 평가 | 지연(lazy) | 지연(lazy) |
| 사용 가능 | 특정 노드 타입에서 기본 활성, 설정으로 켜고 끔 | 항상 사용 가능 |
| 제거 | **LRU 또는 파일 변경 시 자동**, 재시작 시 수동 | **LRU**, `unpersist`로 수동 |

- Databricks는 **자동 disk caching 사용을 권장**하고, Parquet·Delta 데이터의 반복 읽기에는 **Spark cache 대신 disk cache**를 쓰라고 합니다. 성능 모범 사례에서는 "경험칙으로 **Spark caching은 피하라**"고도 말합니다. (쿼리에서 잘못된 위치에 쓰면 **메모리를 모두 소모하고 쿼리를 크게 느리게** 할 수 있음)
- **워커가 해제(decommission)되면 그 워커의 Spark cache가 사라집니다.** 오토스케일링을 켜면 캐시가 **불안정**하고, Spark가 누락된 파티션을 **원본에서 다시 읽어야** 합니다.
- `CACHE TABLE`의 캐시 데이터는 **클러스터의 모든 Spark 세션이 공유**합니다.
- **서버리스 컴퓨트**: `df.cache()`/`df.persist()`는 **Public Preview**(서버리스 환경 버전 5 이상)에서 **동작 방식이 다릅니다.** (즉시 계산해서 세션 동안의 **고정 스냅샷**으로 저장, `persist()`는 저장 수준 인자를 무시) 반면 `CACHE TABLE`, `spark.catalog.cacheTable()`, `df.checkpoint()` 등은 **지원되지 않고 예외가 발생**합니다. 문서마다 서술이 달라 보이는 부분이 있으니 사용하는 클라우드·워크스페이스 기준으로 확인하세요.

# 6. Result cache (쿼리 결과 캐시)

## 6-1. 어디서 동작하나

- **Databricks SQL의 SQL 웨어하우스를 통한 모든 쿼리**에 적용되는 **클러스터 단위** 결과 캐시입니다. 일반 all-purpose 클러스터의 모든 쿼리에 자동 적용되는 캐시가 **아닙니다.**
- 두 가지로 나뉩니다.

| | Local result cache | Remote result cache |
|---|---|---|
| 저장 위치 | 클러스터의 **메모리** | **워크스페이스 시스템 데이터**로 영속화 |
| 적용 | SQL 웨어하우스 | **서버리스 전용** |
| 수명 | **클러스터 수명 또는 캐시가 찰 때까지**, 그리고 **24시간** | **24시간** |
| 재시작 | **정지·재시작하면 비워짐** | **정지·재시작해도 유지** |
| 공유 | 해당 클러스터 | **워크스페이스의 모든 웨어하우스가 공유** |

- 조회 순서: 먼저 **local 캐시**, 그다음 필요하면 **remote 캐시**, 둘 다 없을 때만 **쿼리를 실행**합니다.
- remote 캐시에 접근하려면 **웨어하우스가 실행 중**이어야 합니다.
- remote result cache는 **ODBC/JDBC 클라이언트와 SQL Statement API**로 보낸 쿼리에 사용할 수 있습니다.
- 두 캐시 모두 **기저 테이블이 갱신되면 무효화**됩니다. 성능 모범 사례의 조건: **쿼리가 결정적(deterministic)** 이고 **기저 데이터가 Delta 형식이며 변하지 않았을 때** 결과가 캐시에서 바로 반환됩니다.

## 6-2. 적중하지 않는 쿼리: 시간에 따라 달라지는 함수

- 성능 모범 사례: 결과 캐시를 활용하려면 **`= NOW()` 같은 조건을 쓰지 않는 결정적 쿼리**를 만드세요.
- 대시보드 캐시 문서: **`current_timestamp()` 및 비슷한 함수**는 **쿼리 결과 캐시를 무효화**해 캐시 갱신을 유발합니다. (이 함수들은 대시보드 수준 캐시는 무효화하지 않음)
- 문서가 `rand()`를 결과 캐시의 사례로 **직접 언급하는 곳은 찾지 못했습니다.** 다만 비결정적 함수(`rand()`, `now()` 등)가 쿼리 최적화와 캐시 사용을 제한한다는 설명은 다른 문서(행 필터/컬럼 마스크 성능)에 있습니다. 결과 캐시 적중 여부는 **직접 테스트**로 확인하세요.

```sql
-- 변경 전: 실행할 때마다 값이 달라져 결과 캐시를 활용하기 어려움
SELECT COUNT(*) FROM main.sales.orders
WHERE order_ts >= current_timestamp() - INTERVAL 1 DAY;

-- 변경 후: 날짜를 고정 값(파라미터/리터럴)으로 전달
SELECT COUNT(*) FROM main.sales.orders
WHERE order_ts >= TIMESTAMP '2026-05-05 00:00:00';
```

> `current_date()`처럼 하루 단위로 고정되는 함수가 결과 캐시에서 어떻게 취급되는지는 공식 문서에서 확인하지 못했습니다. 직접 반복 실행해서 적중 여부를 확인하세요.

## 6-3. 끄기 (테스트·벤치마크 전용)

```sql
SET use_cached_result = false;   -- 세션 단위. 시스템 기본값은 TRUE
```

- 공식 문서는 **테스트나 벤치마크에서만** 쓰라고 합니다.
- 결과 캐시를 끄더라도 **Disk cache는 그대로** 동작하므로, 벤치마크에서 두 번째 실행이 더 빠르게 나올 수 있습니다. 공정한 비교를 하려면 **캐시 상태(cold/warm)를 구분**해서 측정하세요.

## 6-4. 일치 조건에 대해

"쿼리 구문이 **완전히 동일**해야 적중한다"는 설명이 흔한데, 이번에 확인한 공식 문서에는 **일치 규칙(공백, 주석, 대소문자 허용 여부 등)이 상세히 기술되어 있지 않았습니다.** 문서에는 "쿼리가 다시 제출되고 기저 테이블이 변하지 않았다면 결과를 재사용할 수 있다", "쿼리 결과 캐시는 **SQL 쿼리를 검사**한다"까지만 나옵니다. 정확한 일치 규칙은 직접 테스트가 필요합니다.

# 7. UI cache와 대시보드 캐시

## 7-1. Databricks SQL UI cache

- **사용자별**로 **쿼리 결과와 SQL 편집기 시각화**를 캐시합니다.
- 사용자가 **SQL 쿼리 또는 레거시 SQL 대시보드를 처음 열 때** 가장 최근 결과를 보여줍니다. **예약 실행 결과도 포함**됩니다.
- **최대 7일** 수명입니다. 캐시는 **내 계정의 Databricks 파일시스템**에 있고, 지우고 싶으면 **쿼리를 다시 실행**하면 옛 결과가 캐시에서 제거됩니다.
- **기저 테이블이 갱신되면 캐시가 무효화**됩니다.
- **AI/BI 대시보드에는 적용되지 않습니다.**

## 7-2. AI/BI 대시보드 캐시 (가장 오해가 많은 부분)

| 항목 | 내용 (공식 문서) |
|---|---|
| 수명 | **24시간**, **최선 노력(best-effort)** — 캐시를 만들지 못하는 경우도 있음 |
| 조회 순서 | **대시보드 캐시 → (없으면) 일반 쿼리 결과 캐시** |
| **신선도** | 대시보드 캐시는 데이터가 바뀌어도 **최대 24시간 지난 결과를 반환할 수 있고, 자동으로 무효화·갱신되지 않음** |
| 쿼리 결과 캐시와의 차이 | 쿼리 결과 캐시는 **오래된 데이터를 반환하지 않음**(기저 데이터 변경 시 무효화) |
| 갱신 방법 | **대시보드 스케줄**이 확실한 방법. **파이프라인 단계에서 데이터를 갱신한다고 대시보드 캐시가 갱신되지는 않음** |
| 웨어하우스 | 캐시에서 제공하면 **웨어하우스가 시작되지 않음** |
| 캐시 단위 | **공유 데이터 권한**으로 게시: 모든 뷰어가 **공유 캐시**. **초안이거나 개별 권한**으로 게시: **사용자별 캐시** |
| 파라미터·필터 | 24시간 내 **같은 파라미터/필터를 쓴 적이 없으면 쿼리 재실행** |
| 브라우저 처리 | 데이터셋이 **100K행·100MB 이하**면 필터·집계를 **브라우저에서** 처리 (쿼리 기록에 안 남음) |

- 대시보드 첫 화면을 빠르게 하려면 **공유 데이터 권한으로 게시한 대시보드에 스케줄**을 거는 것이 좋습니다. 스케줄 실행은 데이터셋 SQL을 실행해 **쿼리 결과 캐시를 채워** 초기 로딩을 개선합니다.
- 결과를 **미리 계산해서 저장**하고 싶다면 **데이터셋 구체화(materialization)** 를 쓰라고 안내합니다.

# 8. 흔한 설명과 공식 문서의 차이

| 자주 보이는 설명 | 공식 문서 |
|---|---|
| Disk cache는 **백그라운드에서** 자동 저장 | **자동으로 저장**한다고만 하고 "백그라운드" 표현은 확인 못 함 |
| "용량 관리: 변경된 파일의 캐시만 무효화" | 그건 **일관성(무효화)** 설명. **용량**은 **로컬 SSD의 최대 절반**, **LRU 제거**, `maxDiskUsage` 등 설정 |
| Result cache는 **모든 쿼리/클러스터**에 적용 | **SQL 웨어하우스**를 통한 쿼리에 적용되는 캐시 |
| Local result cache는 **클러스터 수명과 동일** | **클러스터 수명 또는 캐시가 찰 때까지**, 그리고 **24시간** 수명 |
| Remote result cache는 **컨트롤 플레인**에 저장 | 문서는 **"워크스페이스 시스템 데이터로 영속화"** 라고 표현. **서버리스 전용**, **워크스페이스의 모든 웨어하우스가 공유** |
| 쿼리가 **완전히 같아야** 적중 | 이 표현을 문서에서 확인하지 못함. "쿼리를 검사한다"까지만 기술 |
| `CURRENT_TIMESTAMP()`, `RAND()` 쓰면 **작동 안 함** | 문서에는 `current_timestamp()` 등이 **쿼리 결과 캐시를 무효화**한다고 나옴. **`rand()`는 직접 언급을 못 찾음** |
| UI cache는 **Databricks 대시보드**에 적용, **7일** | **Databricks SQL UI cache**(SQL 편집기·**레거시 SQL 대시보드**)만 7일. **AI/BI 대시보드는 별도의 24시간 캐시** |
| UI cache는 화면 렌더링 보조 | **사용자별 쿼리 결과 캐시**이며, **테이블이 갱신되면 무효화** |
| `CACHE TABLE`은 수동 관리 부담과 메모리 문제 | 맞음. 추가로 **워커 해제 시 소실**, **서버리스에서 `CACHE TABLE` 미지원** |

# 9. 상황별 진단

| 증상 | 의심할 캐시 | 확인·대응 |
|---|---|---|
| 클러스터 재시작 후 **첫 실행이 느림** | **Disk cache**, **Local result cache**가 비워짐 | 정상(재워밍 필요). 반복 실행 후 비교 |
| 서버리스 웨어하우스를 **재시작해도 결과가 빠름** | **Remote result cache**(24시간) | 정상 |
| 데이터를 갱신했는데 **대시보드 숫자가 안 바뀜** | **AI/BI 대시보드 캐시(24시간)** | **대시보드 스케줄**을 걸거나, 캐시로 **최대 24시간 지난 결과가 보일 수 있음**을 전제로 설계 |
| SQL 편집기에서 **옛 결과가 먼저 보임** | **Databricks SQL UI cache**(≤7일) | 쿼리 재실행 시 옛 결과 제거 |
| 같은 쿼리인데 **캐시가 안 맞음** | 비결정적 함수 (`current_timestamp()` 등) | 시간 값을 **고정 값/파라미터**로 전달 |
| 벤치마크 결과가 **들쭉날쭉** | Result cache, Disk cache | `SET use_cached_result = false`(테스트 전용), cold/warm 구분 |
| 같은 테이블을 **반복해서 읽는데 느림** | Disk cache 미적용 | **SSD 캐시 가속 워커 타입** 선택, 설정 확인 |
| **중간 결과**를 여러 번 재사용 | Spark cache 또는 Delta 테이블 | 서버리스는 `CACHE TABLE` 미지원 → 중간 결과를 **Delta 테이블로 저장** |

# 10. 체크리스트

- [ ] 지금 쿼리는 **어떤 컴퓨트**(클러스터 / SQL 웨어하우스 / 서버리스)에서 도는가? 해당 컴퓨트에 **어떤 캐시가 있는가**?
- [ ] **재시작하면 비워지는 캐시**(Disk, Local result)와 **유지되는 캐시**(Remote result, 대시보드)를 구분했는가?
- [ ] 대시보드 데이터의 **신선도 요구**가 24시간보다 짧다면 **스케줄**을 걸었는가?
- [ ] 결과 캐시를 노린 쿼리에서 `current_timestamp()` 등 **비결정적 함수**를 뺐는가?
- [ ] `CACHE TABLE`을 쓰고 있다면 **정말 필요한가**? (Disk cache 우선, 서버리스는 미지원)
- [ ] 벤치마크는 **캐시를 구분**하고 `use_cached_result`를 **테스트 용도로만** 껐는가?

# 11. 한 줄 정리

- 캐시는 **층층이** 있다: **Disk cache**(파일 사본, 로컬 SSD) < **Result cache**(쿼리 결과, SQL 웨어하우스) < **UI/대시보드 캐시**(화면용 결과).
- **Disk cache**는 파일 변경을 **자동 감지**하고 **로컬 SSD 최대 절반 + LRU**로 관리된다. Spark cache보다 **Disk cache를 권장**한다.
- **Result cache**는 **SQL 웨어하우스** 기능이다. local은 메모리, remote는 **서버리스 전용·24시간·워크스페이스 공유**이고, 기저 테이블이 갱신되면 **무효화**된다.
- **UI cache는 최대 7일(SQL 편집기·레거시 대시보드)**, **AI/BI 대시보드 캐시는 24시간이며 데이터가 바뀌어도 자동 갱신되지 않는다.** 둘을 혼동하지 말 것.
- `current_timestamp()` 같은 비결정적 함수는 **결과 캐시를 무효화**한다. **일치 규칙은 직접 테스트**한다.

# 관련 용어 글
Disk cache · Spark cache(CACHE TABLE) · SQL 쿼리 결과 캐시(Local/Remote) · Databricks SQL UI cache · AI/BI 대시보드 캐시

## 🔗 참고
- [Optimize performance with caching on Databricks](https://docs.databricks.com/aws/en/optimizations/disk-cache)
- [Query caching](https://docs.databricks.com/aws/en/sql/user/queries/query-caching)
- [Dataset optimization and caching (AI/BI dashboards)](https://docs.databricks.com/aws/en/dashboards/caching)
- [USE_CACHED_RESULT](https://docs.databricks.com/aws/en/sql/language-manual/parameters/use_cached_result)
- [CACHE TABLE](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-aux-cache-cache-table)
- [Serverless compute limitations](https://docs.databricks.com/aws/en/compute/serverless/limitations)
- [Best practices for performance efficiency](https://docs.databricks.com/aws/en/lakehouse-architecture/performance-efficiency/best-practices)
