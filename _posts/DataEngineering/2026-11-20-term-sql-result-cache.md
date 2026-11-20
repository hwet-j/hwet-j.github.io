---
title:  "[DE/용어] SQL 쿼리 결과 캐시 (Local / Remote result cache)"
excerpt: "SQL 웨어하우스가 쿼리의 최종 결과를 보관해 같은 쿼리를 다시 계산하지 않게 하는 캐시"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, Cache, Result Cache, SQL Warehouse]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[Query caching](https://docs.databricks.com/aws/en/sql/user/queries/query-caching)    
[USE_CACHED_RESULT](https://docs.databricks.com/aws/en/sql/language-manual/parameters/use_cached_result)    
[Dataset optimization and caching](https://docs.databricks.com/aws/en/dashboards/caching)
{: .notice--warning}

# 0. 이 글을 읽기 전에

**Disk cache**가 "원본 파일 읽기"를 줄인다면, **Result cache**는 "계산 자체를 건너뛰는" 캐시입니다. 이 글의 내용은 **Databricks SQL의 SQL 웨어하우스** 기준입니다.

# 1. 한 줄 정의

**쿼리 결과 캐시**는 **SQL 웨어하우스를 통한 쿼리의 결과를 보관**해 두었다가, 같은 쿼리가 다시 들어오고 **기저 테이블이 바뀌지 않았다면** 계산·스캔 없이 **저장된 결과를 반환**하는 캐시입니다. 위치에 따라 **Local**과 **Remote** 두 가지가 함께 동작합니다.

# 2. Local vs Remote

| 항목 | Local result cache | Remote result cache |
|---|---|---|
| 저장 위치 | 클러스터의 **메모리** | **워크스페이스 시스템 데이터**로 영속화 |
| 적용 | SQL 웨어하우스 | **서버리스 전용** |
| 수명 | **클러스터 수명 또는 캐시가 찰 때까지**, 그리고 진입 시점부터 **24시간** | 진입 시점부터 **24시간** |
| 웨어하우스 정지·재시작 | **비워짐** | **유지됨** |
| 공유 범위 | 해당 클러스터 | **워크스페이스의 모든 웨어하우스** |
| 접근 조건 | — | **웨어하우스가 실행 중**이어야 함 |
| 클라이언트 | — | **ODBC/JDBC 클라이언트, SQL Statement API** 쿼리에 사용 가능 |

# 3. 조회 순서

```
쿼리 도착 → ① local cache 확인 → ② (없으면) remote cache 확인 → ③ 둘 다 없으면 쿼리 실행
```

- 쿼리가 실제로 **실행될 때만** 아래의 Disk cache와 원격 스토리지가 사용됩니다.

# 4. 무효화와 조건

- **기저 테이블이 갱신되면 두 캐시 모두 무효화**됩니다. 쿼리 결과 캐시는 **오래된 데이터를 반환하지 않습니다.**
- 성능 모범 사례: **쿼리가 결정적(deterministic)** 이고 **기저 데이터가 Delta 형식이며 변하지 않았을 때** SQL 웨어하우스가 결과를 캐시에서 바로 반환합니다.
- **`= NOW()` 같은 조건**은 피하세요. 대시보드 문서: **`current_timestamp()` 및 비슷한 함수**는 **쿼리 결과 캐시를 무효화**해서 캐시 갱신을 유발합니다.
- **같은 쿼리**의 판단 기준은 문서에 **상세하게 나와 있지 않습니다.** "쿼리가 다시 제출되고 기저 테이블이 변하지 않았다면 재사용할 수 있다", "쿼리 결과 캐시는 **SQL 쿼리를 검사**한다"는 설명까지만 확인했습니다. 공백·주석 차이 등이 어떻게 취급되는지는 **직접 테스트**하세요.
- `rand()` 등 다른 비결정적 함수가 결과 캐시에서 어떻게 처리되는지는 **직접적인 문서 설명을 찾지 못했습니다.** 적중 여부를 실측하세요.

# 5. 끄기

```sql
SET use_cached_result = false;   -- 세션 단위. 시스템 기본값은 TRUE
```

- **테스트·벤치마크에서만** 쓰라고 공식 문서가 권고합니다.
- 켜져 있을 때의 예 (문서 예시): 같은 `SELECT count(1) FROM T;`를 반복하면 **이전 결과를 재사용**합니다.

# 6. 예시로 이해하기

```sql
-- ① 첫 실행: 계산, 결과 저장
SELECT region, SUM(amount) AS total FROM main.sales.orders GROUP BY region;

-- ② 같은 쿼리를 다시 실행 + 테이블 변경 없음 → 캐시에서 즉시 반환

-- ③ 그 사이에 main.sales.orders 가 INSERT/UPDATE 됨 → 캐시 무효화 → 다시 계산
```

# 7. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "모든 클러스터의 모든 쿼리에 적용된다" | **SQL 웨어하우스** 기능 |
| "Local은 클러스터 수명만 본다" | **캐시가 찰 때까지**이며 **24시간** 수명도 있음 |
| "Remote는 컨트롤 플레인에 저장된다" | 문서는 **"워크스페이스 시스템 데이터로 영속화"** 라고 표현, **서버리스 전용** |
| "테이블이 안 바뀌면 영원히 유지" | **24시간** 수명 |
| "결과 캐시 때문에 오래된 데이터가 나온다" | 쿼리 결과 캐시는 **무효화로 오래된 데이터를 반환하지 않음**. 오래된 결과를 볼 수 있는 건 **AI/BI 대시보드 캐시** |

# 8. 한 줄 정리

- 쿼리 결과 캐시 = **SQL 웨어하우스의 쿼리 최종 결과 캐시**. local(메모리)과 remote(서버리스 전용, 영속)가 **24시간** 수명으로 함께 동작한다.
- **기저 테이블이 갱신되면 무효화**, `current_timestamp()` 같은 함수는 **캐시를 무효화**한다.
- 끄는 `SET use_cached_result = false`는 **테스트 전용**.

# 관련 용어
Disk cache, Databricks SQL UI cache, AI/BI 대시보드 캐시

## 🔗 참고
- [Query caching](https://docs.databricks.com/aws/en/sql/user/queries/query-caching)
- [USE_CACHED_RESULT](https://docs.databricks.com/aws/en/sql/language-manual/parameters/use_cached_result)
- [Dataset optimization and caching](https://docs.databricks.com/aws/en/dashboards/caching)
- [Best practices for performance efficiency](https://docs.databricks.com/aws/en/lakehouse-architecture/performance-efficiency/best-practices)
