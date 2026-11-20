---
title:  "[DE/용어] Lakeflow Jobs의 For each task"
excerpt: "입력 목록의 항목마다 task를 반복 실행하는 기능: 구성, 입력 전달, 동시성"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, Lakeflow Jobs, Workflow]

toc: true
toc_sticky: true
published: true

date: 2026-11-19
last_modified_at: 2026-11-19
---

{% raw %}

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-19)    
[Use a For each task to run another task in a loop](https://docs.databricks.com/aws/en/jobs/tasks/for-each)    
[Troubleshoot and repair job failures](https://docs.databricks.com/aws/en/jobs/repair-job-failures)
{: .notice--warning}

# 0. 이 글을 읽기 전에

Databricks Jobs(Lakeflow Jobs)는 **노트북, SQL, Python 스크립트 같은 task들을 순서·의존성에 맞게 실행**하는 서비스입니다. 이 글은 그중 **"목록의 항목마다 같은 task를 반복"** 하는 For each task를 다룹니다. 월별 청크를 처리하는 상황에서 "노트북 하나를 12번 호출"하는 구조를 떠올리면 됩니다.

# 1. 한 줄 정의

**입력 배열의 각 항목마다 nested task를 한 번씩 실행**하는 Lakeflow Jobs의 task 유형입니다. 예: `["2026-01", "2026-02", ...]` 목록을 주고, 월마다 적재 노트북을 실행.

# 2. 왜 쓰는가

for 문을 코드로 쓰는 대신 **Jobs UI 수준에서** 반복을 정의하면 다음 이점이 있습니다.

| | 노트북 안에서 for/스레드 | For each task |
|---|---|---|
| 반복별 실행 이력 | 직접 로그로 관리 | **반복별 표**로 확인 (UI) |
| 반복별 재시도 | 코드로 구현 | task의 **retry 설정**으로 지정 |
| 반복별 알림 | 직접 구현 | task 알림 설정 |
| 동시 실행 수 제어 | `max_workers` 등 코드 | **Concurrency** 설정 |
| 구현 난이도 | 낮음 | 설정이 늘지만 운영 가시성이 높음 |

# 3. 구조

```
Job
 ├─ [task A] 입력 목록 만들기 (예: 청크 목록 → task value)
 │
 ├─ [For each task]  ← 입력 배열을 받음, Concurrency 설정
 │     └─ [nested task] 노트북: 청크 하나를 읽고 쓴다   ← 입력 항목마다 1회 실행
 │
 └─ [task C] 결과 집계/알림 (For each 에 의존)
```

- For each task는 **For each task 본체 + nested task** 두 개를 정의해야 합니다.
- nested task는 표준 task 유형 중 하나이며, **For each task를 중첩할 수는 없습니다.**
- UI에서는 For each 노드 안에 nested task 노드가 들어 있는 형태로 보이고, 두 노드를 클릭해서 전환합니다.
- 반복(iteration)은 **동시에** 실행될 수 있습니다.

# 4. 입력을 전달하는 세 가지 방법

입력은 **객체 배열**이고, 각 객체가 반복 1회에 전달됩니다.

## 4-1. JSON 배열을 직접 입력

Inputs 텍스트 박스에 JSON 배열을 직접 적습니다. 키-값 쌍, 문자열/숫자/불리언, 임의의 복잡한 JSON 객체를 담을 수 있습니다.

```json
[
  {"label": "2026-01", "from": "2026-01-01", "to": "2026-02-01"},
  {"label": "2026-02", "from": "2026-02-01", "to": "2026-03-01"}
]
```

nested task에서는 `{{input}}`(항목 전체) 또는 `{{input.<key>}}`(필드)로 참조합니다. 예를 들어 `[{"table": "sales"}, {"table": "orders"}]`를 입력하면 `{{input.table}}`로 접근합니다.

## 4-2. 앞선 task가 만든 task value 참조

```python
# 앞선 task (이름: build_chunks)
dbutils.jobs.taskValues.set(key="chunks", value=chunk_list)
```

For each task의 Inputs에는 이렇게 적습니다.

```
{{tasks.build_chunks.values.chunks}}
```

→ **동적으로** 청크 목록이 정해질 때(예: 오늘 기준 최근 N개월) 유용합니다.

## 4-3. Job 파라미터 참조

```
{{job.parameters.chunks}}
```

## 입력 크기 제한 (공식 문서)

| 방식 | 제한 |
|---|---|
| UI의 Inputs 텍스트 박스에 직접 입력 | **5,000자** |
| task value 참조로 전달되는 값 | **48 KB** (참조 문자열 자체는 5,000자 이내) |
| Job 파라미터 값 | **10,000자** |

48 KB를 넘는 큰 입력이 필요하면, 큰 설정 파일을 가리키는 **조회(lookup)** 를 전달하는 방법을 문서가 안내합니다. 또한 **제어 테이블 기반(metadata-driven) Job** 튜토리얼도 제공합니다.

# 5. nested task 안에서 입력 사용하기

nested task가 노트북이라면 **노트북 파라미터에 `{{input}}`을 매핑**하고 노트북에서 읽습니다.

```python
import json

chunk = json.loads(dbutils.widgets.get("chunk"))    # 파라미터 "chunk" 에 {{input}} 을 지정한 경우
label, from_dt, to_dt = chunk["label"], chunk["from"], chunk["to"]

# → 이 청크 하나에 대해서만 읽기 + replaceWhere 쓰기
```

# 6. 주요 설정

| 설정 | 설명 |
|---|---|
| **Concurrency** | 동시에 실행할 반복 수. **기본값은 1** |
| 재시도(Retries) | 일반 task의 고급 설정(`max_retries`, `min_retry_interval_millis` 등) |
| 알림 | task 시작/성공/실패 알림 |

> ⚠️ **Concurrency 기본값이 1**이라서 아무 설정을 안 하면 **반복이 한 번에 하나씩 순차 실행**됩니다. 병렬로 돌리려면 값을 올리세요. 올릴수록 Oracle 연결 수·클러스터 사용량·동시 쓰기 충돌 가능성도 함께 늘어납니다.
{: .notice--warning}

# 7. 결과를 다운스트림에서 쓰기

- 다운스트림 task는 **For each task를 의존성**으로 지정할 수 있습니다.
- **nested task를 직접 의존성으로 지정할 수는 없습니다.**
- 대신 nested task가 각 반복에서 설정한 **task value를 "반복 인덱스 순서의 하나의 리스트"로 읽을 수 있습니다.** → 청크별 성공/건수를 모아서 집계하는 데 유용합니다.
- 입력을 만들어 주는 업스트림 task가 **비활성화(disabled)** 되어 있으면 For each가 실패합니다.

# 8. 실행과 모니터링

- Job 실행은 일반 Job과 같습니다.
- 다른 점은 For each task의 실행 이력이 **반복(iteration)별 표**로 보인다는 것입니다. 어떤 청크가 성공/실패했는지 반복 단위로 확인합니다.

# 9. 언제 쓰고 언제 안 쓰나

| 상황 | 추천 |
|---|---|
| 청크가 수십 개 이하, 노트북 하나로 충분 | 노트북 내 **스레드 병렬**이 단순 |
| 청크별 **개별 재시도·알림·이력**이 필요 | **For each** |
| 실패한 청크만 사람이 확인 후 **다시 실행**하고 싶음 | For each + **Repair run** (멱등한 쓰기 필수) |
| 입력 목록이 매우 큼 | 입력 제한(5,000자/48 KB/10,000자) 확인, lookup 방식 고려 |

# 10. 확인되지 않은 점 (직접 테스트 권장)

- 반복 중 일부가 실패했을 때 **For each task 전체의 상태**가 어떻게 되는지, **Repair 시 실패한 반복만** 다시 실행되는지 아니면 For each 전체가 다시 실행되는지는 이번에 공식 문서에서 확인하지 못했습니다. 상세는 공식 문서의 "Run and monitor a job with a For each task" 등을 확인하세요.
- 어느 쪽이든 안전하도록 **각 반복을 멱등하게(replaceWhere)** 만들어 두면 전체가 다시 실행돼도 문제가 없습니다.

# 11. 한 줄 정리

- For each = **목록의 항목마다 nested task를 반복 실행**(예: 월 목록 → 월마다 적재).
- 입력은 **JSON 배열 / task value / job parameter**, 참조는 `{{input}}` / `{{input.key}}`.
- **Concurrency 기본값 1**(순차), 입력 크기 제한이 있다. nested task는 For each를 중첩할 수 없다.
- 청크별 이력·재시도·알림이 필요할 때 적합하며, **멱등한 쓰기**와 함께 써야 안전하다.

# 관련 용어
Repair run, 청크(Chunk), 멱등성, replaceWhere

## 🔗 참고
- [Use a For each task to run another task in a loop](https://docs.databricks.com/aws/en/jobs/tasks/for-each)
- [Troubleshoot and repair job failures](https://docs.databricks.com/aws/en/jobs/repair-job-failures)
{% endraw %}
