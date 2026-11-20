---
title:  "[DE/용어] spark.task.maxFailures"
excerpt: "task가 몇 번 연속 실패해야 Spark가 job을 포기하는지, 그리고 이 설정으로 해결되지 않는 것"

categories:
  - DataEngineering
tags:
  - [Glossary, Spark, Retry, Configuration]

toc: true
toc_sticky: true
published: true

date: 2026-11-19
last_modified_at: 2026-11-19
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-19)    
[Spark configuration](https://spark.apache.org/docs/latest/configuration.html)    
[What are ACID guarantees on Databricks?](https://docs.databricks.com/aws/en/lakehouse/acid)
{: .notice--warning}

# 0. 이 글을 읽기 전에

Spark 작업이 실패했을 때 로그에 `Task ... failed 4 times` 같은 문구를 보신 적이 있다면 이 설정과 관련이 있습니다. 먼저 Spark의 실행 단위를 간단히 짚겠습니다.

## Spark 실행 단위 (개념)

```
Job (쓰기/액션 1회)
 └─ Stage (셔플 경계로 나뉘는 단계)
     └─ Task (파티션 1개를 처리하는 최소 실행 단위)  ← 이 글의 주인공
```

- DataFrame의 **파티션 하나 = task 하나**입니다. (예: JDBC 읽기 파티션 12개 → 읽기 stage의 task 12개)
- task들은 워커(executor)에서 병렬로 실행됩니다.
- **task 하나가 실패하면 Spark는 그 task를 다시 시도**합니다. 이 재시도 한도를 정하는 것이 `spark.task.maxFailures`입니다.

# 1. 한 줄 정의

**같은 task가 연속으로 몇 번 실패하면 job을 포기할지** 정하는 Spark 설정입니다.

# 2. 공식 문서의 정의 (핵심)

| 항목 | 내용 |
|---|---|
| 기본값 | **4** |
| 의미 | 특정 task가 **연속으로** 이 횟수만큼 실패하면 job을 포기 |
| 허용 재시도 횟수 | **이 값 − 1** (기본값이면 재시도 3회) |
| 다른 task의 실패 | **합산되지 않음** (여러 task에 흩어진 실패 총합은 job 실패를 일으키지 않음) |
| 성공하면 | 그 task의 **실패 카운트가 초기화**됨 |
| 허용 범위 | 1 이상 |

# 3. 예제로 이해하기 (기본값 4)

| task | 시도 결과 | 최종 |
|---|---|---|
| 1월 task | 실패 → 실패 → 성공 | ✅ 성공 (연속 2회 실패 후 성공 → 카운트 초기화) |
| 2월 task | 실패 → 실패 → 실패 → 실패 | ❌ **연속 4회 실패 → job 포기(실패)** |
| 3월 task | 실패 → 성공 → 실패 → 성공 | ✅ 성공 (중간에 성공하면 카운트 초기화, 연속 아님) |
| 4월 task | 한 번에 성공 | ✅ |

- 2월 task 하나가 연속 4회 실패한 순간, **1월·3월·4월이 성공했더라도 job은 실패**합니다.
- 반면 1월(2회)+3월(1회)+... 처럼 **서로 다른 task의 실패가 많아도** 각각이 4번 연속에 이르지 않으면 job은 실패하지 않습니다.

# 4. 어디서 설정하나

공식 문서의 설정 설명: `spark.task.maxFailures` 같은 **Spark 런타임 제어용 속성**은 **`spark-submit` 같은 시작 시점 옵션과, 프로그램에서 설정하는 방식 모두** 가능하다고 안내합니다.

```
# 예: 클러스터의 Spark config 항목 (한 줄에 "키 공백 값")
spark.task.maxFailures 8
```

- Databricks에서는 보통 **클러스터의 Spark config**에 위와 같이 지정합니다.
- 실행 중인 세션에서 `spark.conf.set(...)`으로 바꿔도 **이미 시작된 앱에는 적용되지 않을 수 있습니다.** 앱(클러스터) 시작 시점에 지정하는 것이 안전합니다.
- 서버리스 같은 compute에서는 **설정할 수 있는 Spark 속성이 제한**될 수 있으니 사용하는 compute에서 가능한지 확인하세요.

# 5. 값을 올리면 도움이 되는 경우

- Oracle 같은 외부 DB의 **일시적 오류**(네트워크 순단, 일시적 세션 한도 초과, 짧은 락 대기 등)
- 워커 노드 **일시적 문제**(스팟 인스턴스 회수 등으로 task만 다시 시도하면 되는 경우)

# 6. 값을 올려도 소용없는 경우 (중요)

| 원인 | 왜 소용없나 |
|---|---|
| **영구적 오류** (SQL 문법 오류, 존재하지 않는 테이블, 권한 없음) | 몇 번을 다시 해도 똑같이 실패 → 시간만 낭비 |
| **데이터 자체의 문제** (타입 변환 불가 값 등) | 재시도해도 같은 데이터를 읽어 같은 결과 |
| **"일부만 성공"시키고 싶음** | 이 설정은 **task 재시도**일 뿐. 끝내 실패하면 job 전체 실패 → Delta 커밋 없음 |

특히 마지막이 핵심입니다. **`spark.task.maxFailures`를 아무리 올려도 "한 task가 끝내 실패하면 쓰기 전체가 롤백된다"는 구조는 바뀌지 않습니다.** 그 구조는 Delta의 원자성 때문입니다. (→ Delta 트랜잭션 글) 부분 성공이 필요하면 **청크 단위로 쓰기를 쪼개야** 합니다. (→ 청크 글)

# 7. 값을 올릴 때의 부작용

- **외부 DB 부하 증가**: 재시도는 같은 쿼리를 Oracle에 **다시 보냅니다.** 실패 원인이 DB 과부하라면 재시도가 부하를 더 키울 수 있습니다.
- **실패 감지 지연**: 영구 오류도 그만큼 오래 시도한 뒤에야 포기합니다.
- **중복 읽기 비용**: 큰 파티션을 반쯤 읽다 실패해서 처음부터 다시 읽으면 시간 낭비가 큽니다.

# 8. 세 가지 재시도 층위 구분

| 층위 | 설정/기능 | 단위 | 동작 |
|---|---|---|---|
| **① Spark task 재시도** | `spark.task.maxFailures` | task(파티션) | 같은 task를 다시 시도, 연속 실패 한도 초과 시 job 실패 |
| ② Jobs task 재시도 | Job task의 `max_retries` 등 | Job의 task 전체 | 실패한 task(노트북 전체)를 처음부터 다시 실행 |
| ③ Repair run | 사람이 실행 | 실패 task + 의존 task | 실패 지점부터 복구 |

- ①은 **Spark 엔진 내부**, ②③은 **Databricks Jobs(오케스트레이션) 수준**입니다.
- ①로 해결되지 않으면 ②, ②로도 안 되면 ③ 순으로 올라간다고 생각하면 됩니다.
- ②③은 task를 **처음부터** 다시 실행하므로 **멱등한 쓰기**가 전제입니다. (→ 멱등성 글)

# 9. 확인하는 방법

- Spark UI → 해당 Job → Stage → **Tasks 표**에서 task별 **Attempt**(시도 번호)와 **Failed tasks** 수, 에러 메시지를 확인합니다.
- 한 task만 반복 실패한다면 해당 **파티션(예: 특정 월)의 데이터나 쿼리 문제**일 가능성이 큽니다.

# 10. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "4면 총 4번 재시도" | 허용 재시도 = **값 − 1** (총 시도는 4번, 재시도는 3번) |
| "task들의 실패를 합쳐서 4번이면 job 실패" | **합산 안 됨**. 한 task의 **연속** 실패만 센다 |
| "성공한 적이 있어도 계속 누적" | 성공하면 **카운트 초기화** |
| "값을 키우면 일부만 적재된다" | 끝내 실패하면 **전체 롤백**. 부분 성공은 청크로 |
| "Jobs의 max_retries와 같은 것" | **다른 층위**. 이 값은 Spark task 단위 |

# 11. 한 줄 정리

- `spark.task.maxFailures`는 **한 task가 연속 몇 번 실패하면 job을 포기할지**(기본 4, 재시도 3회) 정한다.
- **서로 다른 task의 실패는 합산되지 않고, 성공하면 카운트가 초기화**된다.
- **일시적 오류**에는 도움이 되지만 **영구 오류**나 **부분 적재**에는 도움이 안 된다.
- 올리면 **DB 부하와 지연**이 늘 수 있다.
- 부분 성공은 이 설정이 아니라 **청크 단위 쓰기**로 해결한다.

# 관련 용어
Delta 트랜잭션(ACID), 청크(Chunk), JDBC 읽기 파티션, Repair run, 멱등성

## 🔗 참고
- [Spark configuration](https://spark.apache.org/docs/latest/configuration.html)
- [What are ACID guarantees on Databricks?](https://docs.databricks.com/aws/en/lakehouse/acid)
