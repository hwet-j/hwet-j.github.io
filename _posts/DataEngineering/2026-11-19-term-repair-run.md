---
title:  "[DE/용어] Repair run (실패한 task만 재실행)"
excerpt: "실패한 Job을 처음부터 다시 돌리지 않고 실패 지점부터 복구하는 방법과 주의점"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, Lakeflow Jobs, Retry]

toc: true
toc_sticky: true
published: true

date: 2026-11-19
last_modified_at: 2026-11-19
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-19)    
[Troubleshoot and repair job failures](https://docs.databricks.com/aws/en/jobs/repair-job-failures)
{: .notice--warning}

# 0. 이 글을 읽기 전에

Databricks Job은 여러 task가 의존성으로 이어진 구조일 수 있습니다. 이 글은 **중간 task가 실패했을 때, 성공한 부분은 그대로 두고 실패한 부분부터 다시 실행**하는 기능을 다룹니다.

# 1. 한 줄 정의

실패하거나 취소된 **멀티 task Job**에서, **실패한 task와 그에 의존하는 task만** 다시 실행하는 기능입니다. 성공한 task는 다시 돌지 않습니다.

# 2. 왜 쓰는가: 시간·자원 절약

```
Job: [A 추출 ✅ 2시간] → [B 변환 ❌ 실패] → [C 적재 (B 때문에 skip)]

[Run now 로 처음부터]  A(2시간) + B + C  → 이미 성공한 A 를 또 실행
[Repair run]           B + C 만 실행        → A 는 건너뜀 ✅
```

공식 문서: "성공한 task와 그에 의존하는 성공 task는 다시 실행되지 않으므로, 실패한 Job 실행에서 복구하는 데 필요한 **시간과 자원을 줄인다.**"

# 3. 사용 순서 (UI 기준)

공식 문서의 흐름을 그대로 풀면 다음과 같습니다.

## 3-1. 실패 원인 찾기

1. 사이드바에서 **Jobs & Pipelines** → Job 이름 클릭
2. **Runs** 탭의 **matrix 뷰**에서 task별 성공/실패 이력을 확인 (task는 실패했거나, **의존하는 task가 실패해서 skip**된 상태일 수 있음)
3. 실패한 task에 마우스를 올리면 시작/종료 시각, 상태, 소요 시간, 클러스터 정보, (경우에 따라) 에러 메시지를 봄
4. task를 클릭해 **Task run details**에서 출력·에러·메타데이터 확인

## 3-2. 원인 고치기 (문서의 예시)

| 원인 | 조치 |
|---|---|
| task 설정 문제 | **Edit task**로 수정 후 저장 |
| 클러스터 자원 부족 (예: 인스턴스 부족) | job cluster 대신 공유 all-purpose 클러스터 사용 고려, 워커 수·인스턴스 유형 변경, 다른 클러스터로 Swap, 관리자에게 쿼터 증설 요청 |
| **최대 동시 실행 수 초과** | 다른 실행 완료까지 대기하거나, **Maximum concurrent runs** 값 변경 |
| **업스트림 문제** (예: 외부 데이터 소스 장애) | 외부 문제가 해결된 뒤 repair 사용 |

## 3-3. Repair 실행

1. 실패한 실행의 링크(또는 matrix 뷰의 실패 항목)를 클릭 → **Job run details**
2. **Repair run** 클릭 → 재실행될 **실패 task와 의존 task 목록**이 나옴
3. 필요하면 **파라미터를 추가/수정** (여기서 입력한 값은 기존 값을 **덮어씀**. 이후 repair에서 원래 값으로 되돌리려면 키·값을 비움)
4. 다시 **Repair run** 클릭
5. 끝나면 matrix 뷰에 **repair 실행용 새 열**이 생기고, 빨간 task가 초록으로 바뀜

# 4. 동작 규칙

- 실패한 task는 **처음부터** 다시 실행됩니다. (중간부터 이어서가 아님)
- Repair 전에 **Job/task 설정을 바꿀 수 있고**, 재실행되는 task는 **바뀐 현재 설정**으로 돌아갑니다. (예: 노트북 경로, 클러스터 설정을 고치고 repair)
- Repair는 task의 **실행 상태**로 판단하며, **비활성(disabled) 상태가 기준이 아닙니다.** 비활성 task를 강제로 실행하려면 repair 요청의 `rerun_tasks`에 포함해야 합니다.
- **task가 2개 이상인 Job에서만** 지원됩니다. 단일 task Job은 **Run now**로 다시 실행합니다.

# 5. ⚠️ 가장 중요한 주의: 멱등성은 보장되지 않는다

공식 문서의 경고입니다. (의역)

> Lakeflow Jobs는 task를 멱등하게 만들어 주지 않는다. task가 실패 전에 일부 출력을 이미 썼다면 repair로 재실행할 때 **데이터가 중복될 수 있다.** 안전한지는 task 로직에 달려 있다. repair 전에 각 task가 무엇을 쓰는지 검토하고, 중복이 문제라면 **append 대신 overwrite나 merge** 같은 멱등한 연산을 써라.

## 위험한 예 vs 안전한 예

```python
# ❌ repair 로 재실행되면 같은 데이터가 또 추가될 수 있음
df.write.mode("append").saveAsTable("main.l0.orders")

# ✅ 몇 번 재실행돼도 해당 청크 범위는 한 벌
(df.write.format("delta").mode("overwrite")
   .option("replaceWhere", chunk_cond)
   .saveAsTable("main.l0.orders"))
```

## Repair 전 점검 질문

- 실패한 task가 **어디까지 썼는가?** (외부 테이블, 파일, API 호출)
- 다시 실행하면 **중복·누락**이 생기는가?
- 이 task의 쓰기는 **overwrite/merge**처럼 멱등한가?

# 6. Repair vs 다른 재실행 방법

| 방법 | 무엇을 다시 실행 | 언제 |
|---|---|---|
| **Repair run** | 실패한 task + 의존 task만 | 멀티 task Job에서 중간 실패 |
| **Run now** | Job을 **처음부터** | 단일 task Job, 또는 전체 재실행이 필요 |
| task의 **자동 재시도**(`max_retries`) | 실패한 task를 **자동으로** 즉시/대기 후 다시 | 일시적 오류 |
| **continuous Job의 자동 재시도** | Job 전체를 지수 백오프로 | 연속 실행 Job (재시도 정책은 사용 불가) |

- continuous Job에서는 연속 실패가 임계값을 넘으면 **지수 백오프**로 재시도하고, 상태 창에 연속 실패 횟수·정상 구간·다음 재시도까지의 시간이 표시됩니다. **Restart run**으로 활성 실행을 취소하고 재시도 기간을 초기화한 뒤 새로 시작할 수 있습니다.

# 7. 다른 방법으로 실행하기

- **UI**: 위 순서
- **CLI**:

```bash
databricks jobs repair-run <RUN_ID> --rerun-all-failed-tasks
```

- **API**: Jobs API의 Repair a job run 엔드포인트 (`rerun_tasks` 등으로 대상 task 지정)

# 8. For each task와 함께 쓸 때

청크를 For each task로 처리했을 때, 반복 중 일부가 실패하면 repair로 복구하게 됩니다.

- repair가 **실패한 반복만** 다시 실행하는지, For each 전체를 다시 실행하는지는 이번에 공식 문서에서 명확히 확인하지 못했습니다. 상세 동작은 문서를 확인하세요.
- **어느 쪽이든 안전하도록 각 반복의 쓰기를 멱등(`replaceWhere`)** 으로 만들어 두는 것이 정답입니다.

# 9. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "Repair하면 실패한 task가 중간부터 이어서 실행된다" | **처음부터** 다시 실행 |
| "Repair는 중복을 알아서 막아 준다" | **멱등성은 보장되지 않음**. 사용자 로직 책임 |
| "단일 task Job도 Repair 된다" | **2개 이상** task만. 단일은 Run now |
| "설정을 고치려면 Job을 새로 돌려야 한다" | repair 전에 **설정을 수정**할 수 있고 재실행 task는 현재 설정 사용 |
| "비활성 task도 자동으로 repair 대상" | 상태 기준이며, 비활성 task는 `rerun_tasks`에 **명시**해야 함 |

# 10. 한 줄 정리

- Repair run = **실패한 task와 그 하위 task만 다시 실행**해서 시간과 자원을 아낀다. (처음부터 실행)
- 순서: **원인 파악 → 원인 수정(필요 시 설정/파라미터 변경) → Repair run**.
- **2개 이상 task Job**만 지원. 단일 task는 Run now.
- **멱등성은 보장되지 않는다.** `append` 대신 **`replaceWhere`/`MERGE`** 로 안전하게.

# 관련 용어
For each task, 멱등성, replaceWhere, 청크(Chunk)

## 🔗 참고
- [Troubleshoot and repair job failures](https://docs.databricks.com/aws/en/jobs/repair-job-failures)
