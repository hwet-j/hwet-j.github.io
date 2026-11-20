---
title:  "[DE/용어] Deletion vectors"
excerpt: "행을 지우거나 바꿀 때 파일 전체를 다시 쓰지 않고 '표시'만 해 두는 기능"

categories:
  - DataEngineering
tags:
  - [Glossary, Delta Lake, Deletion Vectors, Performance]

toc: true
toc_sticky: true
published: true

date: 2026-11-19
last_modified_at: 2026-11-19
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-19)    
[Deletion vectors in Databricks](https://docs.databricks.com/aws/en/delta/deletion-vectors)    
[Row-level concurrency](https://docs.databricks.com/aws/en/optimizations/isolation/row-level-concurrency)
{: .notice--warning}

# 0. 이 글을 읽기 전에

"Delta에서 행 하나를 지우면 내부에서 무슨 일이 일어나지?"를 설명합니다. **Delta 트랜잭션(ACID)** 글의 "Delta 테이블 = 데이터 파일 + 트랜잭션 로그" 설명을 먼저 보시면 좋습니다.

# 1. 한 줄 정의

DELETE/UPDATE/MERGE 때 **행이 들어 있는 Parquet 파일 전체를 다시 쓰는 대신**, 그 행이 변경(삭제)됐다는 사실을 **메타데이터에 표시**해 두는 기능입니다. 읽을 때 그 표시를 적용해서 "현재 상태"를 계산합니다.

# 2. 먼저 알아야 할 배경: Parquet 파일은 수정하지 않는다

Delta 테이블의 데이터는 **Parquet 파일**에 저장되는데, Parquet 파일은 한 번 쓰면 **그 안의 행 하나만 고치는 것이 불가능한 구조**에 가깝습니다. (개념 설명) 그래서 행 하나를 지우려면 일반적으로:

```
① file_001.parquet (행 100,000개) 전체를 읽는다
② 지울 행 1개를 뺀 99,999개로 file_001' 를 새로 쓴다
③ 로그에 "file_001 삭제, file_001' 추가" 를 커밋한다
```

행 1개를 지우는데 **10만 행을 다시 쓰는** 셈입니다. 공식 문서도 "deletion vectors가 없으면 행 하나를 수정할 때 그 레코드가 든 파일 전체를 다시 써야 한다"고 설명합니다.

# 3. Deletion vectors가 하는 일

```
[없을 때]  file_001 (100,000행) ─ 행 1개 삭제 ─> file_001' (99,999행) 를 새로 씀  (무겁다)

[있을 때]  file_001 (100,000행) ─ 그대로 둠
           + deletion vector: "file_001 의 7번째 행은 삭제됨" 이라는 작은 표시만 기록 (가볍다)
```

- 쓰는 쪽: 큰 파일을 다시 쓰지 않고 **작은 표시**만 기록하므로 DELETE/UPDATE/MERGE가 빨라집니다.
- 읽는 쪽: 파일을 읽을 때 **표시를 적용**해서 삭제된 행을 걸러냅니다. 그래서 겉으로 보이는 결과는 같습니다.
- 이런 방식을 **소프트 삭제(soft-delete)** 라고 합니다. 행이 물리적으로 지워진 것이 아니라 논리적으로 가려진 상태입니다.

# 4. 왜 동시성 글에 나올까

Row-level concurrency의 **필수 조건**이 deletion vectors입니다. 변경이 "파일 교체"가 아니라 "행 표시"로 기록되기 때문에, 같은 파일의 서로 다른 행을 고치는 두 작업을 합칠 수 있습니다. 문서: "DBR 14.2 이상에서 deletion vectors가 있는 테이블은 row-level concurrency를 지원한다."

# 5. 버전과 호환성

| 항목 | 조건 |
|---|---|
| 모든 최적화를 포함해 **쓰기** | Databricks Runtime **14.3 LTS 이상** |
| **읽기** | Databricks Runtime **12.2 LTS 이상** |
| Photon 있는 DBR | MERGE/UPDATE/DELETE를 12.2 LTS 이상에서 지원 |
| Photon 없는 DBR | DELETE 12.2 LTS+, UPDATE 14.1+, MERGE 14.3 LTS+ |
| Delta 테이블 기본값 | **명시적으로 활성화**해야 함 (워크스페이스 설정으로 신규 테이블에 자동 활성화될 수 있고, 기본 설정은 리전별로 다름) |

- **프로토콜 업그레이드 경고**: deletion vectors를 켜면 테이블 프로토콜이 업그레이드되어, **deletion vectors를 지원하지 않는 클라이언트는 이 테이블을 읽을 수 없게 됩니다.** 외부 엔진(다른 Spark, 다른 도구)이 읽는 테이블이라면 사전에 호환성을 확인하세요.
- DBR 16.4 LTS 이상에서는 deletion vectors **테이블 기능을 제거(drop)** 해서 다른 클라이언트와 호환시킬 수 있습니다. (DBR 14.3 LTS 이상에서는 레거시 방식으로 가능)

# 5-1. 켜고 끄는 방법

```sql
-- 테이블 생성 시
CREATE TABLE main.l0.orders (
  order_dt DATE, order_id STRING, amount DECIMAL(18,2)
) TBLPROPERTIES ('delta.enableDeletionVectors' = true);

-- 기존 테이블
ALTER TABLE main.l0.orders SET TBLPROPERTIES ('delta.enableDeletionVectors' = true);
```

- materialized view / streaming table은 `ALTER`로 켜거나 끌 수 **없고**, **`CREATE` 문에서 지정**해야 합니다.

# 6. 소프트 삭제를 "진짜 삭제"로 만들기

deletion vectors는 행을 **논리적으로** 가릴 뿐이라, 물리 파일에는 삭제된 행이 **그대로 남아 있습니다.** 파일을 실제로 다시 쓰려면 다음 중 하나를 합니다.

| 방법 | 설명 |
|---|---|
| `OPTIMIZE` | 파일을 재작성(compaction)하면서 반영 |
| `REORG TABLE ... APPLY (PURGE)` | deletion vector 변경이 있는 **모든 데이터 파일을 재작성** |
| auto-compaction이 켜진 쓰기 | 쓰기 중 compaction이 일어나면서 반영 |

- 문서 주의: compaction은 deletion vector에 기록된 변경을 **엄격히 해소한다는 보장이 없습니다.** 대상 파일이 compaction 후보가 아니면 일부 변경이 물리적으로 반영되지 않을 수 있습니다.

## 옛 데이터를 물리적으로 완전히 지워야 할 때 (예: 개인정보 삭제 요청, 스토리지 비용)

purge 후에도 **이전 버전의 파일에는 옛 데이터가 남아 있습니다.** 완전히 지우려면:

1. `REORG TABLE ... APPLY (PURGE)` 실행
2. 그 purge가 끝난 시각 이후를 기준으로 **`VACUUM`** 을 실행해 이전 버전의 파일을 물리적으로 제거

큰 테이블에서 purge 성능을 높이려면 `spark.databricks.delta.reorg.purgeMode`를 `rows`로 설정할 수 있습니다. (기본값 `all`은 모든 Parquet 파일 푸터를 스캔해서 느릴 수 있고, `rows`는 소프트 삭제된 행이 있는 파일만 처리)

# 7. 제약

- Iceberg v2 읽기는 deletion vectors를 지원하지 않습니다. (Iceberg v3는 지원)
- deletion vectors를 쓰는 파일이 있는 테이블은 `GENERATE`로 매니페스트를 만들 수 없습니다. 먼저 `REORG TABLE ... APPLY (PURGE)`를 실행하고, 동시 쓰기가 없는지 확인한 뒤 `GENERATE`를 해야 합니다.
- deletion vectors가 켜진 테이블은 매니페스트의 **증분 생성**도 지원하지 않습니다.

# 8. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "DELETE했으니 데이터가 물리적으로 사라졌다" | deletion vectors가 있으면 **표시만** 된 상태. `REORG ... PURGE` + `VACUUM`까지 해야 물리 삭제 |
| "켜면 모든 도구에서 그대로 읽힌다" | 지원하지 않는 클라이언트는 **읽을 수 없게 됨**(프로토콜 업그레이드) |
| "켜기만 하면 동시성 문제가 해결된다" | row-level concurrency는 **DBR 14.3 LTS+ / 파티션 없음 / deletion vectors** 세 조건이 필요 |
| "UPDATE/MERGE에는 효과가 없다" | DELETE뿐 아니라 UPDATE, MERGE도 가속 |

# 9. 한 줄 정리

- Parquet 파일은 일부 행만 고치기 어려워서, 보통 행 하나를 바꿔도 파일 전체를 다시 써야 한다.
- Deletion vectors는 **"이 행은 삭제됨" 표시만 기록**해서 DELETE/UPDATE/MERGE를 빠르게 하고, 읽을 때 표시를 적용한다.
- **Row-level concurrency의 필수 조건**이다.
- 켜면 **프로토콜이 올라가서** 미지원 클라이언트가 읽지 못할 수 있다. 활성화는 `delta.enableDeletionVectors` 속성.
- 물리 삭제는 `OPTIMIZE` / `REORG ... PURGE` 후 `VACUUM`.

# 관련 용어
Row-level concurrency, 격리 수준, Delta 트랜잭션(ACID)

## 🔗 참고
- [Deletion vectors in Databricks](https://docs.databricks.com/aws/en/delta/deletion-vectors)
- [Row-level concurrency](https://docs.databricks.com/aws/en/optimizations/isolation/row-level-concurrency)
