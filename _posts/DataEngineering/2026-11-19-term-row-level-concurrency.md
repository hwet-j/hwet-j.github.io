---
title:  "[DE/용어] Row-level concurrency"
excerpt: "같은 파일의 서로 다른 행을 건드리는 동시 쓰기를 충돌 없이 처리하는 기능"

categories:
  - DataEngineering
tags:
  - [Glossary, Delta Lake, Concurrency, Deletion Vectors]

toc: true
toc_sticky: true
published: true

date: 2026-11-19
last_modified_at: 2026-11-19
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-19)    
[Row-level concurrency](https://docs.databricks.com/aws/en/optimizations/isolation/row-level-concurrency)    
[Liquid clustering](https://docs.databricks.com/aws/en/delta/clustering)    
[Deletion vectors](https://docs.databricks.com/aws/en/delta/deletion-vectors)
{: .notice--warning}

# 0. 이 글을 읽기 전에

"월별로 나눠서 동시에 쓰는데 왜 충돌이 나지?", "서로 다른 행을 고치는데 왜 실패하지?"에 대한 답입니다. **Delta 트랜잭션(ACID)**, **격리 수준** 글을 먼저 보시면 좋습니다.

# 1. 한 줄 정의

동시 쓰기들이 **같은 데이터 파일 안의 서로 다른 행**을 수정·삭제할 때, 충돌을 **행 단위로 감지해서 자동으로 해결**하는 기능입니다.

# 2. 문제 상황: Delta의 충돌은 "파일" 단위

Delta 테이블은 여러 행을 하나의 **파일**에 담아 저장합니다. 행 하나를 수정하려면(기본적으로) 그 행이 든 **파일 전체를 새로 쓰고**, 옛 파일은 "삭제됨"으로 표시합니다.

```
file_001.parquet  = [행1, 행2, 행3, ... 행100]

작업 A: 행5 를 UPDATE      → file_001 을 새로 써서 교체하려 함
작업 B: 행90 을 DELETE     → file_001 을 새로 써서 교체하려 함
```

A와 B는 **서로 다른 행**을 건드리지만 **같은 파일(file_001)** 을 교체하려 합니다. 먼저 커밋한 쪽이 file_001을 "삭제"로 바꿔 놓으면, 나중에 커밋하는 쪽은 "내가 읽은 파일이 이미 없어졌다"는 이유로 **충돌(ConcurrentDeleteReadException 등)** 이 됩니다.

공식 문서도 같은 설명을 합니다. "row-level concurrency가 없으면, 서로 다른 행을 바꾸더라도 **같은 데이터 파일을 수정**하는 두 작업은 충돌할 수 있다. 그래서 같은 테이블에 동시에 MERGE/UPDATE/DELETE를 돌리는 작업이 실패한다."

# 3. 해결: 행 단위로 변경을 추적

Row-level concurrency는 **Deletion vectors**(행 변경을 파일 재작성 대신 메타데이터로 표시하는 기능)를 바탕으로, **같은 파일 안에서 서로 다른 행을 바꾼 경우는 충돌로 보지 않고 자동으로 합쳐줍니다.**

```
작업 A: 행5 변경 표시   ┐
                        ├─ 파일은 그대로, 변경 표시만 기록 → 서로 다른 행이므로 둘 다 성공
작업 B: 행90 삭제 표시  ┘
```

같은 행을 동시에 바꾸면 여전히 충돌합니다.

# 4. 켜지는 조건 (자동)

다음 세 가지를 **모두** 만족하면 자동으로 활성화됩니다.

1. Databricks Runtime **14.3 LTS 이상**
2. 테이블이 **파티션되어 있지 않음**
3. 테이블에 **Deletion vectors가 활성화**되어 있음

- 파티션된 테이블은 row-level concurrency를 쓸 수 없습니다. (파티션된 테이블도 deletion vectors를 켜면 **OPTIMIZE와 쓰기 작업 사이의** 충돌은 피할 수 있습니다.)
- **Liquid clustering이 적용된 테이블**은 row-level concurrency를 허용합니다. 공식 문서는 파티션 테이블을 liquid clustering으로 바꿀 때의 이점으로 "liquid clustering 테이블은 row-level concurrency를 허용해서 쓰기 충돌이 줄어든다"고 설명합니다.
- 구버전: DBR 13.3 LTS에서는 deletion vectors가 필요하고, liquid clustering 테이블은 자동으로 켜지는 레거시 동작을 합니다.

# 5. 충돌 표 (row-level concurrency가 있는 테이블)

여기서 INSERT는 **테이블을 읽는 서브쿼리가 없는 append**를 뜻합니다.

| 작업 쌍 | WriteSerializable (기본) | Serializable |
|---|---|---|
| INSERT + INSERT | 충돌 없음 | 충돌 없음 |
| INSERT + UPDATE/DELETE/MERGE | 충돌 없음 | **같은 행**을 수정하면 충돌 가능 (실패하는 쪽은 UPDATE/DELETE/MERGE) |
| UPDATE/DELETE/MERGE + UPDATE/DELETE/MERGE | **같은 행**을 수정하면 충돌 가능 | **같은 행**을 수정하면 충돌 가능 |
| INSERT + OPTIMIZE | 충돌 없음 | 충돌 없음 |
| UPDATE/DELETE/MERGE + OPTIMIZE | `ZORDER BY` 사용 시에만 충돌 가능 | 동일 |
| OPTIMIZE + OPTIMIZE | `ZORDER BY` 사용 시에만 충돌 가능 | 동일 |

# 6. row-level concurrency가 없을 때의 표 (예: 파티션된 테이블)

| 작업 쌍 | WriteSerializable (기본) | Serializable |
|---|---|---|
| INSERT + INSERT | 충돌 없음 | 충돌 없음 |
| INSERT + UPDATE/DELETE/MERGE | 충돌 없음 | **충돌 가능** (UPDATE/DELETE/MERGE가 실패) |
| UPDATE/DELETE/MERGE + UPDATE/DELETE/MERGE | **충돌 가능** | **충돌 가능** |
| UPDATE/DELETE/MERGE + OPTIMIZE | deletion vectors 테이블은 충돌 없음(ZORDER BY 제외), 아니면 충돌 가능 | 동일 |

"충돌 가능"이라고 표시된 경우에도 **두 작업이 같은 파일 집합을 건드릴 때만** 실제 충돌이 납니다.

# 7. 파티션으로 충돌 피하기 (공식 문서의 방법)

"테이블을 **작업 조건에 쓰는 컬럼으로 파티셔닝**하면 두 작업의 파일 집합이 서로 겹치지 않게 만들 수 있다"고 안내합니다.

- 예: `UPDATE ... WHERE date > '2010-01-01'` 와 `DELETE ... WHERE date < '2010-01-01'` 는 `date`로 파티셔닝하지 않으면 같은 파일을 건드릴 수 있어 충돌합니다. `date`로 파티셔닝하면 충돌하지 않습니다.
- 파티션 컬럼의 카디널리티가 너무 높으면 하위 디렉터리가 폭증해 성능 문제가 생깁니다.

## 조건에 파티션 필터를 **명시**하기

파티션되어 있어도, 조건이 모호하면 Delta가 "전체 테이블을 읽을 수 있다"고 보고 충돌을 냅니다. 공식 문서의 MERGE 예시입니다.

```python
# 문제: 조건 때문에 테이블 전체를 스캔할 수 있음
deltaTable.alias("t").merge(source.alias("s"),
    "s.user_id = t.user_id AND s.date = t.date AND s.country = t.country")

# 해결: 파티션 필터를 조건에 명시
deltaTable.alias("t").merge(source.alias("s"),
    "s.user_id = t.user_id AND s.date = t.date AND s.country = t.country "
    "AND t.date = '2026-01-15' AND t.country = 'KR'")
```

# 8. 충돌 시 보게 되는 예외

| 예외 | 언제 발생 |
|---|---|
| ConcurrentAppendException | 내가 **읽은** 파티션(비파티션 테이블이면 테이블 어디든)에 다른 작업이 파일을 추가 |
| ConcurrentDeleteReadException | 내가 읽은 파일을 다른 작업이 삭제(DELETE/UPDATE/MERGE가 파일을 재작성) |
| ConcurrentDeleteDeleteException | 같은 파일을 둘이 삭제(예: 동시 compaction 두 개) |
| MetadataChangedException | 다른 트랜잭션이 메타데이터 변경(`ALTER TABLE`, 스키마를 바꾸는 쓰기) |
| ConcurrentTransactionException | 같은 체크포인트의 스트리밍 쿼리를 동시에 여러 번 시작 |
| ProtocolChangedException | 테이블 프로토콜 업그레이드, 여러 작성자가 동시에 테이블을 생성/교체 |

- **ConcurrentAppendException**에 대한 중요한 주의: WriteSerializable에서도 여러 DELETE/UPDATE/MERGE가 INSERT가 추가한 값을 **읽을 수 있으면** 충돌할 수 있고, 이때 **실패하는 쪽은 DELETE/UPDATE/MERGE**입니다. 피하려면 (1) 동시 DELETE/UPDATE/MERGE가 추가된 데이터를 읽지 않게 하거나 (2) 추가된 데이터를 읽을 수 있는 DELETE/UPDATE/MERGE를 **최대 하나**로 제한합니다.
- 메타데이터 변경은 표의 **모든 결과에 대한 예외**입니다. INSERT를 포함한 모든 동시 쓰기를 실패시킬 수 있습니다.

# 9. 제약과 주의

- 복합 타입(struct/array/map) 조건, 비결정적 식, 서브쿼리/상관 서브쿼리 조건은 row-level concurrency가 적용되지 않고 **일반 동시성 규칙**을 따릅니다.
- DBR 14.2에서는 MERGE에 **대상 테이블에 대한 명시적 조건**이 필요합니다.
- 행 단위 충돌 감지는 **총 실행 시간을 늘릴 수 있고**, 동시 트랜잭션이 많으면 충돌 해결보다 **지연 시간을 우선**합니다.
- **identity 컬럼**이 있는 테이블은 동시 트랜잭션을 지원하지 않습니다.
- 충돌이 나면 **읽은 쪽(UPDATE/DELETE/MERGE)만 실패**합니다. 데이터를 읽지 않는 append는 실패하지 않으므로, **재시도 로직은 UPDATE/DELETE/MERGE 쪽에** 둡니다.

# 10. 현실 적용 팁: `replaceWhere`를 동시에 돌릴 때

공식 충돌 표에는 `replaceWhere`(덮어쓰기)가 **별도 행으로 나와 있지 않습니다.** 서로 다른 월에 대한 `replaceWhere`를 동시에 실행했을 때의 충돌 여부는 문서에 명시되지 않으므로, 아래처럼 접근합니다.

1. 테이블 레이아웃을 정한다 (liquid clustering 또는 청크 컬럼 기준 파티셔닝)
2. **소규모로 동시 실행 테스트**
3. 충돌 시 재시도, 동시 실행 수 축소, 최후에는 순차 실행

# 11. 한 줄 정리

- Delta의 충돌은 기본적으로 **파일 단위**. 서로 다른 행이라도 같은 파일이면 부딪힌다.
- Row-level concurrency는 **행 단위**로 감지해 같은 파일의 다른 행 변경을 자동으로 합친다.
- 조건: **DBR 14.3 LTS+ / 파티션 없음 / Deletion vectors 켜짐**. Liquid clustering 테이블은 이 기능을 허용한다.
- 순수 append끼리는 충돌하지 않고, 충돌 시 **실패하는 쪽은 읽는 쪽(UPDATE/DELETE/MERGE)** 이다.
- 파티션 테이블에서는 **조건 컬럼으로 파티셔닝 + 조건에 파티션 필터 명시**로 충돌을 줄인다.

# 관련 용어
Deletion vectors, 격리 수준, 테이블 파티셔닝, Liquid Clustering, replaceWhere

## 🔗 참고
- [Row-level concurrency](https://docs.databricks.com/aws/en/optimizations/isolation/row-level-concurrency)
- [Isolation levels](https://docs.databricks.com/aws/en/optimizations/isolation/isolation-levels)
- [Use liquid clustering for tables](https://docs.databricks.com/aws/en/delta/clustering)
- [Deletion vectors in Databricks](https://docs.databricks.com/aws/en/delta/deletion-vectors)
