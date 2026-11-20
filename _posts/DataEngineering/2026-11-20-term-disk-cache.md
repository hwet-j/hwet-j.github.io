---
title:  "[DE/용어] Disk cache (디스크 캐시)"
excerpt: "원격 스토리지의 Parquet 파일 사본을 워커의 로컬 SSD에 자동 보관해 반복 읽기를 빠르게 하는 기능"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, Cache, Disk Cache, Performance]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[Optimize performance with caching on Databricks](https://docs.databricks.com/aws/en/optimizations/disk-cache)    
[Query caching](https://docs.databricks.com/aws/en/sql/user/queries/query-caching)
{: .notice--warning}

# 0. 이 글을 읽기 전에

클라우드 스토리지(S3 등)에서 파일을 읽는 것은 **로컬 디스크에서 읽는 것보다 느립니다.** 같은 데이터를 계속 읽는 작업이라면 매번 원격에서 가져오는 것이 낭비입니다. 이 글은 그 낭비를 줄이는 **Disk cache**를 다룹니다. 다른 캐시와의 관계는 *Databricks 캐시 총정리* 글을 참고하세요.

# 1. 한 줄 정의

**Disk cache**는 노드가 원격 스토리지에서 **Parquet 데이터 파일을 가져올 때 그 사본을 자동으로 로컬 저장소에 빠른 중간 포맷으로 저장**해 두고, 이후 같은 데이터를 **로컬에서 읽게 하는** Databricks의 기능입니다.

> 예전 이름은 **Delta cache**, **DBIO cache**였습니다. Delta Lake 프로토콜의 일부로 오해되지 않도록 이름을 바꿨고, **Databricks 전용 기능**입니다.
{: .notice--info}

# 2. 왜 필요한가

```
[캐시 없음]  쿼리 1 ─ 원격 스토리지에서 파일 읽기
             쿼리 2 ─ 같은 파일을 또 원격에서 읽기       (매번 느림)

[Disk cache] 쿼리 1 ─ 원격에서 읽고 로컬 SSD에 사본 저장 (Cold)
             쿼리 2 ─ 로컬 SSD에서 읽기                  (빠름, Warm)
```

# 3. 핵심 동작

| 항목 | 내용 |
|---|---|
| 대상 | **모든 Parquet 데이터 파일** (Delta Lake 테이블 포함) |
| 시작 | **첫 읽기 때 자동** 저장 (캐시가 켜져 있을 때) |
| 저장 단위 | **파일**. 쿼리 조건이 달라도 **같은 파일을 읽으면** 캐시를 재사용 |
| 저장하지 못하는 것 | **임의의 서브쿼리 결과** (그건 Spark cache의 영역) |
| 평가 | 지연(lazy) |
| 효과가 큰 경우 | **같은 테이블을 여러 번 읽는** 반복·반복적 쿼리 |

# 4. 데이터가 바뀌면? (일관성)

- 데이터 파일이 **생성·삭제·수정·덮어쓰기**되는 것을 **자동 감지**해서 내용을 갱신합니다.
- 테이블 데이터를 쓰거나 수정·삭제해도 **명시적으로 캐시를 비울 필요가 없고**, 오래된 항목은 **자동으로 무효화되어 제거**됩니다.
- 따라서 `INSERT`/`UPDATE`/`DELETE`로 **바뀐 파일의 캐시만** 무효화되고, 바뀌지 않은 파일의 캐시는 유지됩니다. (Delta에서 행 변경이 파일 교체로 이어지기 때문 — 개념 설명)

# 5. 용량과 수명

| 항목 | 내용 |
|---|---|
| 사용 용량 | 워커의 로컬 SSD가 제공하는 공간의 **최대 절반** |
| 가득 차면 | **LRU**(가장 오래 쓰지 않은 것부터) 제거 |
| 클러스터 재시작 | 캐시가 **비워짐** (SQL 웨어하우스에서는 Local result cache와 같은 수명) |
| 오토스케일링 | 노드가 사라지면 그 노드의 캐시도 사라짐 (Spark cache와 마찬가지로 **불안정**할 수 있음) |

```ini
# 클러스터 생성 시 Spark 설정
spark.databricks.io.cache.maxDiskUsage 50g            # 노드당 캐시 데이터용 디스크
spark.databricks.io.cache.maxMetaDataCache 1g         # 노드당 캐시 메타데이터용 디스크
spark.databricks.io.cache.compression.enabled false   # 캐시 데이터를 압축 저장할지
```

# 6. 켜기·끄기·확인

```python
spark.conf.get("spark.databricks.io.cache.enabled")             # 현재 설정
spark.conf.set("spark.databricks.io.cache.enabled", "false")    # 끄기 ("true"로 켜기)
```

- 끄는 것은 **이미 저장된 데이터를 삭제하지 않습니다.** 대신 쿼리가 **캐시에 새 데이터를 넣거나 캐시에서 읽는 것**을 막습니다.
- 가장 쉬운 사용법은 클러스터를 만들 때 **SSD가 달린 캐시 가속(cache-accelerated) 워커 타입**을 고르는 것입니다. 이 인스턴스는 **자동으로 최적 구성**되고, 지원하는 노드에서는 **기본으로 켜져 있습니다.**
- **SQL 웨어하우스와 DBR 14.2 이상**에서는 `CACHE SELECT` 명령이 **무시**되고 **개선된 디스크 캐시 알고리즘**이 대신 쓰입니다.

# 7. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "쿼리 결과를 저장한다" | **파일 사본**을 저장. 결과 저장은 Result cache |
| "테이블을 수정하면 캐시를 직접 비워야 한다" | **자동 감지·무효화** |
| "재시작해도 남는다" | 로컬 디스크 캐시는 **재시작 시 비워짐** |
| "Spark cache가 더 좋다" | 문서는 Parquet·Delta 반복 읽기에 **Disk cache를 권장** |
| "Delta Lake의 기능이다" | **Databricks 전용**. 그래서 이름을 Delta cache에서 바꿈 |

# 8. 한 줄 정리

- Disk cache = **원격 Parquet 파일의 사본을 워커 로컬 SSD에 자동 보관**하는 기능. 파일 단위, 파일 변경 시 **자동 무효화**.
- 용량은 **SSD의 최대 절반**, **LRU 제거**, 재시작하면 비워진다. **SSD 캐시 가속 워커 타입**을 고르는 것이 가장 쉽다.
- 쿼리 결과가 아니라 **파일**을 저장하므로 조건이 달라도 같은 파일이면 효과가 있다.

# 관련 용어
Spark cache(CACHE TABLE), SQL 쿼리 결과 캐시, 클러스터

## 🔗 참고
- [Optimize performance with caching on Databricks](https://docs.databricks.com/aws/en/optimizations/disk-cache)
- [Query caching](https://docs.databricks.com/aws/en/sql/user/queries/query-caching)
- [Best practices for performance efficiency](https://docs.databricks.com/aws/en/lakehouse-architecture/performance-efficiency/best-practices)
