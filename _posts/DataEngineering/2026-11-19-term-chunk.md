---
title:  "[DE/용어] 청크(Chunk)"
excerpt: "큰 적재 작업을 독립적으로 성공·실패·재실행할 수 있는 단위로 쪼개는 설계 방법"

categories:
  - DataEngineering
tags:
  - [Glossary, Data Loading, Parallelism, Design]

toc: true
toc_sticky: true
published: true

date: 2026-11-19
last_modified_at: 2026-11-19
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-19)    
[What are ACID guarantees on Databricks?](https://docs.databricks.com/aws/en/lakehouse/acid)    
[Selectively overwrite data with Delta Lake](https://docs.databricks.com/aws/en/delta/selective-overwrite)    
[For each task](https://docs.databricks.com/aws/en/jobs/tasks/for-each)
{: .notice--warning}

# 0. 이 글을 읽기 전에

> **"청크(chunk)"는 Databricks나 Spark의 공식 기능 이름이 아닙니다.** 이 시리즈에서 쓰는 **설계 용어**(덩어리, 조각이라는 뜻)입니다. 그래서 이 글에는 "공식 문서가 이렇게 말한다"보다 **"공식 문서의 동작 때문에 이런 설계가 필요하다"** 는 설명이 많습니다. 근거가 되는 동작은 **Delta 트랜잭션(ACID)**, **JDBC 읽기 파티션** 글을 참고하세요.
{: .notice--info}

# 1. 한 줄 정의

큰 적재 작업을 **서로 독립적으로 읽고, 쓰고, 성공/실패하고, 다시 실행할 수 있는 작은 단위**로 나눈 것입니다. 예: "2026년 1월 한 달 분량", "주문번호 1~100만".

# 2. 왜 필요한가

## 2-1. 문제: 크게 한 번에 하면 "전부 아니면 전무"

Spark에서 DataFrame 하나를 쓰는 것은 **Delta 커밋 한 번**입니다. 큰 데이터를 한 번에 쓰면, 그중 일부가 실패해도 **전체가 반영되지 않습니다.**

```
[한 번에 쓰기]
  1월 ✓  2월 ✗  3월 ✓  ...  12월 ✓   →  쓰기 1회  →  실패 → 12개월 전부 반영 안 됨
```

## 2-2. 해결: 단위를 쪼개서 각각 따로 쓰기

```
[청크 단위 쓰기]
  1월 → 읽기+쓰기 ✓ (커밋됨)
  2월 → 읽기+쓰기 ✗ (실패, 아무것도 안 바뀜)
  3월 → 읽기+쓰기 ✓ (커밋됨)
  ...
  → 결과: 11개월 반영, 2월만 재실행
```

이사 짐을 **박스 하나씩 따로** 옮기는 것과 비슷합니다(비유). 한 번에 다 옮기다가 하나가 깨지면 전체를 다시 옮겨야 하지만, 박스 단위로 옮기면 깨진 박스만 다시 옮기면 됩니다.

# 3. 청크와 헷갈리기 쉬운 용어

| 용어 | 정체 | 무엇을 나누나 |
|---|---|---|
| **청크 (이 글)** | 설계상의 **작업 단위** | **읽기 + 쓰기 + 성공/실패** 전체를 하나로 묶음 |
| JDBC 읽기 파티션 | Spark가 DB를 병렬로 **읽는** 단위 | 읽기만 나눔 (쓰기는 여전히 1번) |
| Delta 테이블 파티셔닝 | 테이블 **저장** 방식 | 저장 위치만 나눔 |
| Spark 파티션 | 메모리 상의 데이터 조각 | 처리 병렬성 |

핵심 차이: **JDBC 읽기 파티션 12개 ≠ 청크 12개**입니다.
- 읽기 파티션 12개: 읽기는 12갈래로 병렬이지만 **쓰기 커밋은 1번** → 한 곳이 실패하면 전부 실패.
- 청크 12개: 읽기와 쓰기가 **12번 따로** → 한 곳이 실패해도 나머지는 성공.

# 4. 좋은 청크의 조건

| 조건 | 왜 중요한가 | 나쁜 예 |
|---|---|---|
| **서로 겹치지 않음** | 겹치면 같은 행이 두 번 적재되거나 서로 덮어씀 | 1월 청크 `<= 01-31`, 2월 청크 `>= 01-31` |
| **빠짐없이 전체를 덮음** | 빠진 구간은 영영 적재되지 않음 | 월말(31일 23:59:59 이후) 구간 누락 |
| **Delta 컬럼으로 표현 가능** | `replaceWhere`의 조건으로 써야 함 | Oracle ROWID 범위 (Delta 테이블에 없음) |
| **멱등하게 쓸 수 있음** | 실패 후 재실행해도 중복이 없어야 함 | 단순 `append` |
| **적당한 크기** | 너무 작으면 오버헤드, 너무 크면 실패 비용·메모리 부담 | 1일 100건짜리 청크 수천 개 / 1년치 한 청크 |

## 경계를 만드는 안전한 방법: 반열린 구간

```
청크 = [시작, 다음 시작)    즉  col >= 시작  AND  col < 다음시작
```

```
2026-01 : order_dt >= '2026-01-01' AND order_dt < '2026-02-01'
2026-02 : order_dt >= '2026-02-01' AND order_dt < '2026-03-01'
```

- `DATE`든 `TIMESTAMP`든 경계 행이 **정확히 한 청크에만** 들어갑니다.
- `BETWEEN '2026-01-01' AND '2026-01-31'`은 `TIMESTAMP` 컬럼에서 **1월 31일 00:00:01 이후 행을 누락**시킬 수 있어 위험합니다. (경계 포함 방식의 전형적인 함정)

# 5. 청크 단위를 정하는 법

| 기준 | 적합한 경우 | 주의 |
|---|---|---|
| **월** | 데이터가 월 단위로 쌓이고 한 달이 감당 가능한 크기 | 한 달이 너무 크면 메모리·시간 부담 |
| **일** | 하루 증분 배치, 한 달이 너무 큼 | 청크 수가 많아져 오버헤드(Spark job 수) 증가 |
| **키 범위**(예: 번호 1~100만) | 날짜가 없거나 균등 분배가 필요 | Delta 컬럼으로 조건을 표현할 수 있어야 `replaceWhere` 사용 가능 |
| **원천 테이블/파티션** | 원천이 이미 월·일 파티션된 경우 | 원천 파티션 이름과 Delta 조건을 일치시켜야 함 |

감을 잡는 질문:
- 청크 하나가 **실패했을 때 다시 읽는 비용**을 감당할 수 있는가?
- 청크 하나의 **크기가 클러스터 메모리/시간**에 무리가 없는가?
- 동시에 몇 개를 돌리면 **Oracle 연결 수**를 넘지 않는가?

# 6. 구현을 이루는 네 가지 요소

(실제 이름이 아닌 예시 코드입니다.)

## 6-1. 청크 정의: 읽기 조건과 쓰기 조건을 **같은 범위**로

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Chunk:
    label: str          # "2026-01"          (식별/로그용)
    oracle_cond: str    # Oracle 조회용 WHERE
    delta_cond: str     # Delta replaceWhere 용 조건 (위와 동일한 범위!)
```

읽기 조건과 쓰기 조건이 **어긋나면** 사고가 납니다. 예를 들어 읽기는 1~2월을 읽는데 `replaceWhere`는 1월만 지정하면 2월 행 때문에 **오류**가 납니다.

## 6-2. 청크 하나를 처리하는 함수 (읽기 + 쓰기)

```python
def load_chunk(chunk):
    df = spark.read.jdbc(url, f"(SELECT ... FROM SALES.ORDERS WHERE {chunk.oracle_cond}) src",
                         properties=conn_props)
    (df.write.format("delta").mode("overwrite")
       .option("replaceWhere", chunk.delta_cond)
       .saveAsTable("main.l0.orders"))
```

- 이 함수 **하나가 하나의 트랜잭션 단위**입니다. 성공하면 그 청크만 커밋됩니다.

## 6-3. 실행기: 여러 청크를 (병렬로) 돌리고 결과를 모음

```python
ok, failed = [], {}
with ThreadPoolExecutor(max_workers=3) as ex:        # 동시 3개 → 동시 Oracle 연결 약 3개
    futs = {ex.submit(load_chunk, c): c for c in chunks}
    for f in as_completed(futs):
        try:   ok.append(f.result())
        except Exception as e: failed[futs[f].label] = repr(e)[:300]
```

## 6-4. 결과 집계와 후처리

- 성공/실패 청크 목록, 총 건수를 기록하고 task value로 다운스트림에 전달
- 실패가 있으면 **Job을 실패로 끝내서** 알림·재실행을 유도 (성공 청크는 이미 반영됨)

# 7. 청크 상태를 기록하기 (운영에서 유용한 패턴)

청크 수가 많아지면 "무엇이 끝났고 무엇이 남았는지"를 **제어 테이블(control table)** 에 기록하는 방식이 편합니다. (공식 문서도 For each task와 함께 쓰는 "제어 테이블 기반(metadata-driven) Job" 튜토리얼을 안내합니다.)

| chunk_label | status | attempts | rows | last_error | updated_at |
|---|---|---|---|---|---|
| 2026-01 | SUCCESS | 1 | 1,250,000 | | ... |
| 2026-02 | FAILED | 2 | | Connection reset... | ... |
| 2026-03 | PENDING | 0 | | | ... |

- 재실행 시 `status != 'SUCCESS'`인 청크만 대상으로 삼으면 **이미 성공한 청크를 다시 읽지 않습니다.**
- 이 테이블 자체도 업데이트가 멱등하게 되도록(청크 키 기준 MERGE 등) 설계합니다.

# 8. 청크를 병렬로 돌릴 때 고려할 것

| 항목 | 설명 |
|---|---|
| **동시 연결 수** | 동시에 도는 청크 수 ≈ 동시 Oracle 세션 수. DB 한도 확인 |
| **클러스터 자원** | 청크마다 Spark job이 돌아 코어를 나눠 씀. 너무 많이 동시에 돌리면 오히려 느려짐 |
| **동시 쓰기 충돌** | 같은 파일을 건드리는 쓰기끼리 충돌 가능 (→ Row-level concurrency 글). 충돌 시 재시도·동시 수 감소·순차 실행 |
| **재시도 정책** | 일시 오류는 소량의 재시도(+대기), 영구 오류는 빠르게 실패 처리 |
| **빈 청크** | 원천이 0건일 때 기존 데이터를 지울지 유지할지 (replaceWhere는 비면 삭제할 수 있음) |
| **부분 성공의 허용 여부** | 다운스트림이 "12개월 전부"를 요구하면 실패 상태를 전달해 후속 처리를 막아야 함 |

# 9. 청크 방식의 단점과 대안

- **오버헤드**: Spark job이 청크 수만큼 생김. 청크가 너무 작으면 비효율.
- **복잡도**: 코드·상태 관리가 늘어남.
- **원자성 범위 축소**: "전체가 한 번에 반영"이 아니라 **청크 단위로 반영**됨. 전체 원자성이 꼭 필요하면 스테이징 테이블에 모아 한 번에 반영하는 방식을 고려.

# 10. 오픈된 질문 하나

월 청크를 동시에 `replaceWhere`로 쓸 때 충돌이 나는지는 공식 충돌 표에 별도 행이 없어 **문서에 명시되지 않았습니다.** 테이블 레이아웃(liquid clustering/파티셔닝)과 함께 실제 환경에서 소규모로 확인하세요.

# 11. 한 줄 정리

- **청크 = 읽기+쓰기를 한 단위로 묶어 독립적으로 성공·실패·재실행할 수 있게 만든 작업 단위** (공식 용어 아님).
- Spark 쓰기 1회 = Delta 커밋 1회라서 "큰 한 번"은 **전부 아니면 전무**. 청크로 쪼개면 **부분 성공**이 가능.
- 경계는 **반열린 구간**, 읽기 조건과 쓰기 조건은 **같은 범위**, 쓰기는 **멱등(`replaceWhere`)**.
- 병렬도는 **Oracle 연결 수·클러스터 자원·쓰기 충돌**을 보고 정한다.

# 관련 용어
Delta 트랜잭션(ACID), JDBC 읽기 파티션, replaceWhere, 멱등성, For each task, Repair run

## 🔗 참고
- 공식 용어가 아니며, 이 설계의 근거가 되는 동작은 아래 문서를 참고하세요.
- [What are ACID guarantees on Databricks?](https://docs.databricks.com/aws/en/lakehouse/acid)
- [Selectively overwrite data with Delta Lake](https://docs.databricks.com/aws/en/delta/selective-overwrite)
- [Use a For each task to run another task in a loop](https://docs.databricks.com/aws/en/jobs/tasks/for-each)
