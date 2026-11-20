---
title:  "[DE/용어] 격리 수준: WriteSerializable과 Serializable"
excerpt: "동시에 쓰는 작업들이 서로 어떻게 영향을 주는지 정하는 설정을 예제로 이해하기"

categories:
  - DataEngineering
tags:
  - [Glossary, Delta Lake, Isolation Level, Concurrency]

toc: true
toc_sticky: true
published: true

date: 2026-11-19
last_modified_at: 2026-11-19
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-19)    
[Isolation levels (WriteSerializable and Serializable)](https://docs.databricks.com/aws/en/optimizations/isolation/isolation-levels)    
[Row-level concurrency](https://docs.databricks.com/aws/en/optimizations/isolation/row-level-concurrency)
{: .notice--warning}

# 0. 이 글을 읽기 전에

"같은 테이블에 두 개의 Job이 동시에 쓰면 어떻게 되지?"에 대한 답입니다. 먼저 **Delta 트랜잭션과 원자성(ACID)** 글을 읽으면 이해가 쉽습니다. 이 글에서는 용어가 어렵기로 유명한 **직렬화 가능(serializable)** 개념부터 풀어서 설명합니다.

# 1. 한 줄 정의

**격리 수준(isolation level)** 은 Delta 테이블에서 **동시에 실행되는 작업들이 서로 어떻게 상호작용할지**를 정하는 규칙입니다. Delta Lake on Databricks는 두 가지를 지원하고, 기본값은 **WriteSerializable**입니다.

# 2. 핵심 개념 먼저: "직렬화 가능(Serializable)"이란?

> 동시에 실행된 작업들의 결과가, **그 작업들을 한 번에 하나씩 어떤 순서로 실행했을 때의 결과와 같다**면 직렬화 가능하다.

식당 주문으로 비유하면(이해를 돕기 위한 비유입니다), 주방에서 요리사 두 명이 동시에 일했는데 결과가 "한 명이 먼저 만들고 다른 한 명이 나중에 만든 것"과 똑같이 설명된다면 직렬화 가능한 것입니다. 설명이 안 되는 이상한 결과가 나오면 직렬화 불가능입니다.

# 3. 두 가지 수준 비교

| | Serializable | WriteSerializable (기본값) |
|---|---|---|
| 강도 | **가장 강함** | 조금 약함 (snapshot isolation보다는 강함) |
| 무엇을 직렬화 보장? | **커밋된 쓰기 + 모든 읽기** | **쓰기만** (읽기는 아님) |
| 쓰기의 직렬 순서 | **테이블 히스토리 순서와 동일** | 히스토리 순서와 다를 수 있음 |
| 충돌 시 | 나중 작업이 **실패** (재시도 필요) | 순서를 재배치할 수 있으면 **성공** |
| 처리량 | 동시 쓰기 처리량 **감소** | 대부분의 작업에 좋은 균형 |

- 읽기 작업은 **항상 snapshot isolation**을 사용합니다. 쓰기 격리 수준은 "히스토리에 존재한 적 없는 테이블 상태를 읽는 사람이 볼 수 있는가"를 결정합니다.
  - Serializable: 읽는 사람은 **항상 히스토리와 일치하는** 테이블만 본다.
  - WriteSerializable: 읽는 사람이 **Delta 로그에는 존재하지 않는 상태**를 볼 수도 있다.

# 4. 가장 중요한 예제: 동시 DELETE와 INSERT

공식 문서의 시나리오를 숫자와 함께 풀어 봅니다. 테이블 `orders`의 버전 v0에 다음 데이터가 있다고 하겠습니다.

| order_id | status_cd |
|---|---|
| 1 | `CANCEL` |
| 2 | `OK` |

그리고 두 작업이 **동시에 시작**합니다. 둘 다 v0을 읽었습니다.

- **작업 D (DELETE)**: `DELETE FROM orders WHERE status_cd = 'CANCEL'` — 오래 걸리는 작업
- **작업 I (INSERT)**: `INSERT INTO orders VALUES (3, 'CANCEL')` — 금방 끝나는 작업

```
t0  D 시작 (v0 읽음)
t1  I 시작 (v0 읽음)
t2  I 커밋  → v1   (이제 테이블에 order 3 'CANCEL' 존재)
t3  D 커밋 시도 → v2
```

D는 시작할 때 v0을 읽었기 때문에 **I가 넣은 행(order 3)을 보지 못했고, 삭제하지도 않았습니다.** 이때 t3에서 D가 커밋될 수 있는지가 격리 수준에 따라 달라집니다.

| | 결과 |
|---|---|
| **Serializable** | D의 커밋이 **거부**됩니다(충돌 예외). D를 다시 실행해야 합니다. 재실행하면 order 3도 삭제됩니다. |
| **WriteSerializable** | D가 **커밋됩니다.** 결과는 "I가 D보다 **뒤에** 일어난 것처럼" 해석되어 order 3 `CANCEL`이 **남습니다.** 단, 히스토리에는 I(v1)가 D(v2)보다 앞에 기록되어 있어서, 히스토리를 순서대로 재생한 결과와 테이블 내용이 달라집니다. |

즉 WriteSerializable에서는 **"DELETE가 끝난 뒤에도 조건에 맞는 행이 남아 있을 수 있다"** 는 점을 받아들여야 합니다. 대신 충돌로 인한 실패가 적어서 동시 쓰기가 잘 됩니다.

# 5. 언제 Serializable로 바꾸는가

공식 문서는 **대부분의 워크로드에서 기본값(WriteSerializable)을 유지**하라고 합니다. 아래처럼 위 결과를 받아들일 수 없을 때만 Serializable을 고려합니다.

- DELETE가 **커밋 시점에 존재하는 모든 일치 행**을 반드시 지워야 하는 경우
- 다운스트림 프로세스가 **히스토리의 커밋 순서를 재생**해서 테이블 내용을 재구성하는 경우

단, Serializable은 **동시 쓰기 처리량을 낮춥니다.** 또한 "테이블을 읽지 않는 append"도 Serializable에서는 충돌할 수 있습니다.

# 6. 설정하고 확인하기

```sql
-- Serializable 로 변경
ALTER TABLE main.l0.orders SET TBLPROPERTIES ('delta.isolationLevel' = 'Serializable');

-- 기본값(WriteSerializable)으로 되돌림
ALTER TABLE main.l0.orders SET TBLPROPERTIES ('delta.isolationLevel' = 'WriteSerializable');

-- 현재 값 확인
SHOW TBLPROPERTIES main.l0.orders;
```

# 7. 테이블을 "읽지 않는" append란?

WriteSerializable에서 **append(INSERT)는 충돌하지 않는다**고 했는데, 정확히는 **테이블 상태를 읽지 않고 커밋하는 append**를 말합니다. Delta가 읽지 않고 커밋하는 조건은 다음과 같습니다.

1. `INSERT` SQL 또는 **append 모드**로 표현된 로직
2. 대상 테이블을 참조하는 **서브쿼리나 조건식이 없음**

주의할 함정:

```sql
-- "없으면 넣기" 패턴: 조건에 대상 테이블 컬럼을 참조 → MERGE 와 같은 동시성 제약을 받음
INSERT INTO main.l0.orders
SELECT * FROM staged_orders s
WHERE NOT EXISTS (SELECT 1 FROM main.l0.orders t WHERE t.order_id = s.order_id);
```

이런 문장은 `INSERT`처럼 보여도 대상 테이블을 **읽기** 때문에, 단순 append와 달리 충돌할 수 있습니다.

# 8. 충돌 예외와 만났을 때

충돌이 나면 보통 다음 예외 중 하나를 보게 됩니다. (상세는 Row-level concurrency 글 참고)

| 예외 | 의미 |
|---|---|
| ConcurrentAppendException | 내가 **읽은** 범위에 다른 작업이 파일을 추가함 |
| ConcurrentDeleteReadException | 내가 읽은 파일을 다른 작업이 삭제(재작성)함 |
| ConcurrentDeleteDeleteException | 같은 파일을 둘이 삭제하려 함(예: 동시 compaction) |
| MetadataChangedException | 다른 트랜잭션이 테이블 메타데이터(스키마 등)를 변경함 |

# 9. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "Serializable이 항상 더 안전하니 켜자" | 처리량이 낮아지고 충돌이 늘 수 있음. 필요한 경우만 |
| "WriteSerializable은 데이터가 깨질 수 있다" | 깨지는 것이 아니라 "히스토리 순서 재생과 결과가 다를 수 있는" 것 |
| "append는 어떤 수준에서도 안 부딪힌다" | WriteSerializable에서만 그렇다. Serializable에서는 충돌 가능 |
| "격리 수준이 읽기도 바꾼다" | 읽기는 항상 snapshot isolation. 수준은 "읽을 수 있는 상태의 종류"에 영향 |

# 10. 한 줄 정리

- **WriteSerializable(기본)**: 쓰기만 직렬화 보장. 충돌이 적고 처리량이 좋다. 일부 DELETE는 동시에 들어온 INSERT를 못 지울 수 있다.
- **Serializable**: 가장 엄격. 충돌 시 나중 작업이 실패하므로 재시도가 필요하고 처리량이 낮다.
- 대부분은 **기본값 유지**, 위 두 조건(커밋 시점 전부 삭제 / 커밋 순서 재생)일 때만 변경.
- 읽지 않는 순수 append는 WriteSerializable에서 **충돌하지 않는다**.

# 관련 용어
Delta 트랜잭션(ACID), Row-level concurrency, Deletion vectors

## 🔗 참고
- [Isolation levels (WriteSerializable and Serializable)](https://docs.databricks.com/aws/en/optimizations/isolation/isolation-levels)
- [Row-level concurrency](https://docs.databricks.com/aws/en/optimizations/isolation/row-level-concurrency)
