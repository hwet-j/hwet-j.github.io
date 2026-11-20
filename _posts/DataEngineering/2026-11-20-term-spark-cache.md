---
title:  "[DE/용어] Spark cache (CACHE TABLE, cache(), persist())"
excerpt: "DataFrame이나 쿼리 결과를 메모리에 올려 재사용하는 Spark의 캐시와 Databricks에서의 권장 사항"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, Cache, Spark Cache, CACHE TABLE]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[Optimize performance with caching on Databricks](https://docs.databricks.com/aws/en/optimizations/disk-cache)    
[CACHE TABLE](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-aux-cache-cache-table)    
[Serverless compute limitations](https://docs.databricks.com/aws/en/compute/serverless/limitations)
{: .notice--warning}

# 0. 이 글을 읽기 전에

Spark를 써 본 분이라면 `df.cache()`나 `CACHE TABLE`을 알 것입니다. 이 글은 그 **Spark의 캐시**가 Databricks에서 어떻게 취급되는지, **Disk cache와 무엇이 다른지** 설명합니다.

# 1. 한 줄 정의

**Spark cache**는 DataFrame/RDD나 테이블·쿼리 결과를 **메모리(또는 저장 수준에 따라 디스크)에 블록으로 보관**해서 이후 재사용하는 Apache Spark의 캐싱입니다. **직접 코드(또는 SQL)로 지정**해야 합니다.

# 2. 사용 방법

```python
df.cache()                       # 기본 저장 수준으로 캐시 (지연 평가: 처음 사용할 때 채워짐)
df.persist()                     # 저장 수준을 지정할 수 있는 방식
df.unpersist()                   # 해제
```

```sql
CACHE TABLE main.sales.orders;                          -- 즉시 캐시
CACHE LAZY TABLE main.sales.orders;                     -- 처음 사용할 때 캐시
CACHE TABLE recent_orders AS SELECT * FROM main.sales.orders WHERE order_dt >= '2026-05-01';
CACHE TABLE t OPTIONS ('storageLevel' = 'MEMORY_AND_DISK');
```

- 문법: `CACHE [ LAZY ] TABLE table_name [ OPTIONS ( 'storageLevel' [ = ] value ) ] [ [ AS ] query ]`
- 쿼리를 캐시하면 **그 쿼리에 대한 임시 뷰**가 만들어집니다. 원본 파일의 스캔을 줄이는 것이 목적입니다.
- `storageLevel` 값: `NONE`, `DISK_ONLY`, `DISK_ONLY_2`, `MEMORY_ONLY`, `MEMORY_ONLY_2`, `MEMORY_ONLY_SER`, `MEMORY_ONLY_SER_2`, `MEMORY_AND_DISK`, `MEMORY_AND_DISK_2`
- 캐시된 데이터는 **클러스터의 모든 Spark 세션이 공유**합니다.

# 3. Disk cache와의 차이 (공식 문서 표)

| 항목 | Disk cache | Spark cache |
|---|---|---|
| 저장 | 워커 **로컬 파일** | **메모리 블록**(저장 수준에 따름) |
| 대상 | **Parquet 테이블** | **모든 DataFrame/RDD** |
| 시작 | **자동** (첫 읽기) | **수동**, 코드 변경 필요 |
| 제거 | LRU·파일 변경 시 자동 | LRU, `unpersist`로 수동 |
| 사용 가능 | 특정 노드 타입에서 기본 활성 | 항상 사용 가능 |

Spark cache가 **더 낫게** 쓰일 수 있는 경우: **서브쿼리의 임의 결과**나 **Parquet 이외 형식(CSV, JSON, ORC)** 을 저장해야 할 때. 단 공식 문서는 **"경험칙으로 Spark caching은 피하라"** 고 합니다. 잘못된 위치에 쓰면 **메모리를 모두 소모하고 쿼리를 크게 느리게** 할 수 있습니다.

# 4. 주의할 점

- **워커가 해제(decommission)되면 그 워커의 Spark cache가 사라집니다.** 오토스케일링을 켜면 캐시가 **불안정**해지고, Spark가 누락된 파티션을 **원본에서 다시 읽어야** 합니다.
- 메모리를 직접 점유하므로 **캐시가 많으면 다른 작업과 경쟁**합니다.
- 수동 관리라서 **`unpersist`를 빠뜨리면** 계속 메모리를 차지합니다.
- 데이터가 바뀌어도 **자동으로 갱신되지 않습니다.** (Disk cache는 파일 변경을 자동 감지)

# 5. 서버리스 컴퓨트에서는?

| API | 서버리스 동작 (공식 문서) |
|---|---|
| `df.cache()`, `df.persist()`, `df.unpersist()` | **Public Preview**, **서버리스 환경 버전 5 이상**에서 지원. 컴플라이언스 보안 프로필이 켜진 워크스페이스에서는 **사용 불가** |
| `df.checkpoint()` | **지원 안 됨 → 예외** |
| `spark.catalog.cacheTable()`, `uncacheTable()`, `clearCache()` | **지원 안 됨 → 예외** |
| `CACHE TABLE`, `UNCACHE TABLE` | **지원 안 됨 → 예외** |

- 서버리스에서의 `df.cache()`/`persist()`는 **즉시 계산해서 세션 동안의 고정 스냅샷으로 저장**하고, `persist()`는 **저장 수준 인자를 무시**하는 등 클래식과 **동작이 다릅니다.**
- 일부 문서에는 "서버리스에서 DataFrame 캐시 API가 지원되지 않는다"는 이전 서술이 남아 있어 **문서마다 설명이 다를 수 있습니다.** 사용하는 클라우드·워크스페이스 기준으로 확인하세요. 중간 결과를 재사용해야 하면 **Delta 테이블에 저장**하는 방법이 안전합니다.

# 6. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "캐시는 무조건 빨라진다" | 메모리를 소모해 **오히려 느려질 수 있음** |
| "Databricks에서도 `CACHE TABLE`을 기본으로 쓰자" | 문서는 **자동 Disk cache를 권장**, Spark cache는 피하라고 함 |
| "데이터가 바뀌면 캐시도 갱신" | Spark cache는 **자동 갱신 안 됨** |
| "서버리스에서도 `CACHE TABLE`이 된다" | **예외 발생** |

# 7. 한 줄 정리

- Spark cache = **DataFrame·쿼리 결과를 메모리에 올리는 수동 캐시**. 임의의 서브쿼리와 비 Parquet 형식도 담을 수 있지만 **메모리 문제와 불안정성**이 있다.
- Databricks는 **Parquet·Delta 반복 읽기에는 Disk cache**를 권장하고 Spark caching은 **피하라**고 한다.
- **서버리스**에서는 `CACHE TABLE`·`cacheTable`·`checkpoint`가 **예외**, `df.cache()`는 **Preview로 동작이 다르다.**

# 관련 용어
Disk cache, SQL 쿼리 결과 캐시

## 🔗 참고
- [Optimize performance with caching on Databricks](https://docs.databricks.com/aws/en/optimizations/disk-cache)
- [CACHE TABLE](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-aux-cache-cache-table)
- [Serverless compute limitations](https://docs.databricks.com/aws/en/compute/serverless/limitations)
