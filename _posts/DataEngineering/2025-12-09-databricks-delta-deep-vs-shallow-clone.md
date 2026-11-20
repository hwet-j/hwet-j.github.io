---
title:  "[DE/Databricks] Delta Deep Clone vs Shallow Clone: 개념 비교와 소규모 테이블 실측 결과"
excerpt: "두 클론의 차이를 공식 문서로 정리하고, 16개 테이블(50~450MB)을 직접 측정한 결과로 '크기보다 파일 개수가 시간에 영향을 준다'는 결론을 검증"

categories:
  - DataEngineering
tags:
  - [Databricks, Delta Lake, Clone, Deep Clone, Shallow Clone, Benchmark]

toc: true
toc_sticky: true
published: true

date: 2025-12-09
last_modified_at: 2026-11-20
---

업무 중 직접 측정한 결과를 일반화해서 정리했습니다. 테이블명은 모두 **예시**(`tbl_01` ...)입니다.    
공식 문서(AWS 기준)를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[Clone a table on Databricks](https://docs.databricks.com/aws/en/delta/clone) ·
[Shallow clone for Unity Catalog tables](https://docs.databricks.com/aws/en/delta/clone-unity-catalog) ·
[CREATE TABLE CLONE](https://docs.databricks.com/aws/en/sql/language-manual/delta-clone)
{: .notice--warning}

# 1. 상황

운영 테이블을 **테스트·실험·백업용으로 복제**해야 합니다. Delta Lake의 `CLONE`에는 **Deep clone**(데이터까지 복사)과 **Shallow clone**(메타데이터만 복사)이 있습니다. 어느 쪽을 쓸지 정하려고

1. 두 방식의 **개념과 제약**을 정리하고,
2. 크기가 50~450MB인 테이블 16개로 **생성 시간을 직접 측정**했습니다.

# 2. 한눈에 보는 결론

| 항목 | 결과 |
|---|---|
| 개념 | **Deep = 데이터+메타데이터 복사**(독립), **Shallow = 메타데이터만 복사**(소스 파일 참조) |
| 속도 (직접 측정) | 16개 테이블 평균 **Deep 3.7초 / Shallow 2.8초**. 이 크기(≤450MB)에서는 **차이가 몇 초 수준** |
| 시간에 영향을 주는 것 | 이 데이터에서는 **크기보다 파일 개수**. 파일 41~43개 → 409~412개에서 Deep은 약 2.7초 → 5.3초, Shallow는 약 1.9초 → 2.5초 |
| 주의 | **1회씩만 측정**했고 앞쪽 2개 테이블은 이상치로 보임. **GB~TB 규모에는 일반화할 수 없음** |
| 선택 기준 | 이 크기에서는 속도보다 **독립성, VACUUM, 권한, 저장 공간** 같은 기능 차이가 더 중요 |

# 3. 개념 비교 (공식 문서)

| 구분 | Shallow clone | Deep clone |
|---|---|---|
| 복사 대상 | **메타데이터만** (데이터 파일은 복사하지 않고 **소스 파일을 참조**) | **데이터와 메타데이터 모두**, 스트림 메타데이터 포함 |
| 생성 비용 | 컴퓨트·저장 공간을 **적게** 씀 | 데이터를 복사하므로 **비쌈** |
| 소스 의존 | **소스 파일에 의존** | **소스에 의존하지 않음** (독립) |
| 복제 후 변경 | 가능(클론 변경은 소스에 영향 없음) | 가능 |
| 구문 | `SHALLOW CLONE` | `CLONE` 또는 `DEEP CLONE` (기본값) |
| 클라우드 권한 | 클론 디렉터리 + **소스의 데이터 파일**을 읽을 권한 | 클론 디렉터리 읽기/쓰기 권한 |
| 용도(문서 예시) | **프로덕션 테이블 대상 단기 실험** | **보관(archiving), 재해 복구, ML 학습 데이터 재현** |

- 복제되는 메타데이터: 스키마, 파티션 정보, invariants, nullability, `TBLPROPERTIES`. **Deep에서만** 스트림 및 `COPY INTO` 메타데이터도 복제되어, 소스에 쓰던 **스트림을 클론에서 이어서** 실행할 수 있습니다.
- **복제되지 않는 것**: 테이블 설명, 사용자 정의 커밋 메타데이터, **Delta 이력**, Unity Catalog 속성(태그 등). 클론은 **독립된 이력**을 가지므로 소스와 같은 입력으로 time travel이 되지 않습니다.
- 필요한 권한(공통): 소스에 **SELECT**, 새 테이블을 만들면 스키마에 **CREATE**, 기존 테이블을 대체하면 **MODIFY**.
- **스트리밍 테이블과 materialized view는 CLONE의 소스·타깃이 될 수 없습니다.**
- CTAS와 다르게 **파티셔닝, 포맷, invariants, nullability 등을 따로 지정하지 않아도** 메타데이터가 복사됩니다.

## 사용법

```sql
-- Deep clone (기본)
CREATE TABLE main.demo.tbl_copy CLONE main.demo.tbl_src;
CREATE TABLE main.demo.tbl_copy DEEP CLONE main.demo.tbl_src;

-- Shallow clone
CREATE TABLE main.demo.tbl_copy SHALLOW CLONE main.demo.tbl_src;
CREATE TABLE main.demo.tbl_copy SHALLOW CLONE main.demo.tbl_src VERSION AS OF 15;
CREATE TABLE main.demo.tbl_copy SHALLOW CLONE main.demo.tbl_src TIMESTAMP AS OF '2025-12-01';

-- 이미 있는 타깃을 교체 (Unity Catalog의 기존 shallow clone에는 사용 불가)
CREATE OR REPLACE TABLE main.demo.tbl_copy DEEP CLONE main.demo.tbl_src;
```

- CLONE이 끝나면 **한 행짜리 메트릭**이 반환됩니다: `source_table_size`, `source_num_of_files`, `num_copied_files`(**shallow는 0**), `copied_files_size` 등. 테스트 결과 검증에 쓸 수 있습니다.

# 4. 정리표 검증: 정리한 내용 vs 공식 문서

| 정리 내용 | 공식 문서 | 판정 |
|---|---|---|
| Shallow: 메타데이터만 복사, 복사 후 DML 가능 | 메타데이터만 복사. 문서 예시에서 클론에 `UPDATE`를 실행하고 필요하면 소스에 `MERGE`로 반영 | ✅ 일치 |
| Shallow: 생성 속도 빠름, 저장 공간 매우 작음 | "생성 비용이 낮고 저장 공간을 덜 쓴다" | ✅ 일치 (속도는 4장 실측으로도 확인) |
| Shallow: **소스에서 VACUUM하면 사용 불가, 재생성 필요** | **Hive metastore/비UC**: 맞음 (`FileNotFoundException`, `clone`을 `replace`로 다시 실행해서 복구). **Unity Catalog**: **소스에서 `VACUUM`해도 클론이 깨지지 않음** (UC가 클론과 소스 파일의 관계를 추적) | ⚠️ **환경에 따라 다름.** Unity Catalog 테이블이면 문서상 해당 없음. 사용 환경에서 직접 확인 필요 |
| Shallow: Dedicated에서 타깃 변경 시 **소스 MODIFY 필요**, Standard는 불필요 | Dedicated 모드: 타깃에 insert/update 시 **소스 MODIFY** 필요. Standard 모드: **타깃의 MODIFY** 필요(소스 불필요). Databricks는 **Standard 모드 사용을 권장** | ✅ 일치 (문서 확인 + **직접 테스트로 확인**, *UC Shallow clone 접근 모드 권한 테스트* 글 참고) |
| Deep: 소스와 **동일한 저장 공간** 필요 | 데이터를 복사하는 독립 복사본 | ✅ 개념상 일치 |
| Deep: 대용량일수록 복사 시간 소요 | "데이터를 복사하므로 만드는 비용이 큼" | ✅ 문서와 일치. 단 **이번 측정(≤450MB)에서는 크기 영향이 작았음** (5장) |
| Deep: **증분 동기화는 별도 파이프라인 필요** | 기존 타깃에 CLONE을 **다시 실행하면 마지막 클론 이후의 새 메타데이터·데이터만 담은 증분 커밋**이 생김. 문서 예시: 한 달에 한 번 `CREATE OR REPLACE TABLE archive CLONE prod` 실행 | ⚠️ **보강**: 별도 파이프라인이 아니라 **CLONE을 주기적으로 재실행**(스케줄)하면 증분 동기화됨 |
| Deep: 복사 후 소스 MODIFY 불필요 | Deep clone은 소스에 의존하지 않는 독립 복사본 | ✅ 개념상 일치 (문서에 MODIFY를 직접 언급한 부분은 확인 못함) |

# 5. 실측 테스트

## 5-1. 환경과 방법

| 항목 | 내용 |
|---|---|
| 클러스터 | Driver `Standard_D8ds_v5` 1대 + Worker `Standard_D8ds_v5` 2대 |
| 대상 | 크기 50MB ~ 500MB 테이블 **16개** (파일 개수 4 ~ 412개) |
| 측정 | 테이블마다 **Deep와 Shallow를 각 1회** 클론하고 `duration_sec` 기록 (총 32회) |
| 결과 | 32회 모두 `Success` |
| 기록되지 않은 것 | Databricks Runtime 버전, Unity Catalog 여부(관리형/외부), 접근 모드, **실행 순서**, `duration_sec`이 측정하는 구간 |

## 5-2. 결과 (파일 개수 순)

| 테이블 | 파일 수 | 크기(MB) | Deep(초) | Shallow(초) | Deep ÷ Shallow |
|---|---|---|---|---|---|
| tbl_01 | 4 | 106.00 | 7.00 | 6.08 | 1.15 |
| tbl_02 | 41 | 433.06 | 2.85 | 2.29 | 1.24 |
| tbl_03 | 41 | 450.34 | 2.74 | 2.01 | 1.36 |
| tbl_04 | 41 | 450.34 | 3.13 | 3.86 | 0.81 |
| tbl_05 | 42 | 51.01 | 2.52 | 1.98 | 1.28 |
| tbl_06 | 42 | 72.74 | 2.43 | 1.77 | 1.37 |
| tbl_07 | 42 | 123.28 | 2.99 | 6.61 | 0.45 |
| tbl_08 | 42 | 261.38 | 2.81 | 1.99 | 1.42 |
| tbl_09 | 42 | 320.60 | 2.83 | 1.76 | 1.61 |
| tbl_10 | 42 | 448.98 | 2.85 | 1.77 | 1.61 |
| tbl_11 | 43 | 63.01 | 2.50 | 1.85 | 1.35 |
| tbl_12 | 154 | 317.05 | 3.63 | 1.91 | 1.90 |
| tbl_13 | 409 | 249.59 | 5.15 | 2.69 | 1.91 |
| tbl_14 | 409 | 255.06 | 5.28 | 2.36 | 2.24 |
| tbl_15 | 409 | 255.06 | 5.07 | 2.35 | 2.16 |
| tbl_16 | 412 | 61.65 | 5.86 | 2.76 | 2.12 |

## 5-3. 파일 개수 구간별 요약

| 파일 수 | 테이블 수 | 크기 범위(MB) | Deep 평균(초) | Shallow 평균(초) | 비율 |
|---|---|---|---|---|---|
| 41~43 (이상치 2개 제외) | 8 | 51~450 | **2.69** | **1.93** | 1.40배 |
| 154 | 1 | 317 | 3.63 | 1.91 | 1.90배 |
| 409~412 | 4 | 62~255 | **5.34** | **2.54** | 2.10배 |

- 이상치 2개(tbl_04, tbl_07)를 제외한 14개 기준 **Deep 평균 3.55초, Shallow 평균 2.24초**(Deep가 약 1.6배).
- 전체 16개 기준 **Deep 평균 3.73초, Shallow 평균 2.75초**(Deep가 약 1.35배).

## 5-4. 분석: "크기보다 파일 개수가 영향이 크다"

계산한 상관관계(결정계수 R²)입니다. (이상치 제외 14개 기준)

| | 시간 vs 파일 개수 | 시간 vs 크기 |
|---|---|---|
| **Deep** | R² = **0.96** | R² = 0.03 |
| **Shallow** | R² = 0.10 | R² = 0.05 |

- **Deep**: 시간이 **파일 개수와 강하게 비례**하고 크기와는 거의 무관합니다. 같은 41~43개 파일에서 크기가 51MB → 450MB로 **약 9배** 커져도 Deep 시간은 **2.4초 → 2.85초**로 조금만 늘었지만, 파일이 42개 → 409개로 **약 10배** 늘면 **2.7초 → 5.3초**로 크게 늘었습니다.
- **Shallow**: 파일이 늘어도 **거의 평평**합니다(1.76 ~ 2.29초대). 데이터 파일을 복사하지 않기 때문으로 해석됩니다.
- 대략적인 추정: Deep는 파일 하나당 약 **7ms**씩 늘어나는 정도(409~412개 구간 평균과 41~43개 구간 평균의 차이 ÷ 파일 수 차이). **구간 2개·1회 측정으로 만든 거친 추정**입니다.

> 이 분석은 정리해 둔 결론 — "테스트 데이터 기준으로 크기보다 파일 개수가 처리 시간에 더 영향을 준다(Deep에서 더 잘 나타남)" — 과 **일치**합니다.
{: .notice--info}

## 5-5. 이상치와 한계

| 항목 | 내용 |
|---|---|
| tbl_01 | 파일이 **4개뿐인데 Deep 7.00초, Shallow 6.08초**로 가장 느림. 파일 개수 가설과 맞지 않음 |
| tbl_04, tbl_07 | **Shallow가 Deep보다 느림**(0.81, 0.45배). 다른 14개는 모두 Shallow가 더 빠름 |
| 추정 원인 | 목록 앞쪽 테이블이라 **첫 실행 지연(워밍업, 캐시, 클러스터 초기 동작)** 일 가능성. **실행 순서가 기록되지 않아 확인 불가** → 추정 |
| 반복 | **각 조건 1회**라 우연한 변동과 구분할 수 없음 |
| 규모 | **50~450MB**에서만 측정. **GB~TB에서는 데이터 복사량이 지배적**일 것으로 예상되나 **이번에 측정하지 않음** |
| 절대 차이 | 모두 **몇 초 이내**. 이 크기에서는 **속도가 선택을 좌우하지 않음** |
| 환경 | 런타임, UC 여부, 접근 모드 기록 없음 |

## 5-6. 다음에 측정한다면 (재측정 설계)

- 각 조건을 **5회 이상 반복**하고 중앙값을 사용
- **실행 순서를 무작위화**하고 **워밍업 실행**을 1회 버림
- **`num_copied_files`, `copied_files_size`, `source_num_of_files`** 메트릭을 함께 기록해 측정이 의도대로 동작하는지 검증 (Shallow는 `num_copied_files = 0`이어야 함)
- **GB~TB 규모** 테이블과 **파일 개수가 큰(수천~수만)** 테이블을 추가
- 런타임 버전, UC 관리형/외부, 접근 모드, 타깃 저장 공간(`DESCRIBE DETAIL`의 `sizeInBytes`)을 기록

```python
import time

def timed_clone(src, tgt, kind):          # kind: "DEEP" 또는 "SHALLOW"
    spark.sql(f"DROP TABLE IF EXISTS {tgt}")
    t0 = time.perf_counter()
    metrics = spark.sql(f"CREATE TABLE {tgt} {kind} CLONE {src}").collect()[0].asDict()
    elapsed = time.perf_counter() - t0
    return {"kind": kind, "sec": round(elapsed, 3),
            "files": metrics.get("source_num_of_files"),
            "copied_files": metrics.get("num_copied_files")}
# (예시 코드, 실행해서 검증하지 않음)
```

# 6. Unity Catalog에서의 Shallow clone (제약)

| 항목 | 내용 (공식 문서) |
|---|---|
| 상태 | **Public Preview** |
| 런타임 | 관리형 테이블 **DBR 13.3 LTS+**, 외부 테이블 **DBR 14.3 LTS+** |
| 대상 | **관리형 → 관리형**, **외부 → 외부**만 가능. **Delta 테이블만** 가능 |
| `VACUUM` | **소스에서 VACUUM해도 클론이 깨지지 않음.** UC가 소스·클론의 파일 관계를 추적해서 **필요한 파일은 지우지 않음** |
| 제약 | 기존 shallow clone을 `CREATE OR REPLACE`로 **덮어쓸 수 없음**(DROP 후 CREATE), **shallow clone의 shallow clone 불가**, **OpenSharing으로 공유 불가** |
| 권한 독립 | 클론의 소유자가 **소스와 별개로 접근 권한을 통제** |
| 베이스 테이블 삭제 | 관리형은 베이스를 삭제해도 **소스 파일이 지워지기 전까지** 클론이 읽힘. 그 뒤에는 사용 불가 |

# 7. 선택 가이드

| 상황 | 선택 | 이유 |
|---|---|---|
| 운영 테이블로 **짧은 실험·검증** | **Shallow** | 저장 공간·시간이 적고 운영 데이터에 영향 없음 |
| **장기 보관·백업·재해 복구** | **Deep** | 소스와 독립. 증분 동기화 가능 (CLONE 재실행) |
| **소스에서 VACUUM이 잦고** 비-UC 환경 | **Deep** (또는 UC 사용) | 비-UC shallow는 VACUUM 후 깨짐 |
| **ML 학습 데이터 재현** | `VERSION AS OF` + Deep (문서 예시) | 특정 버전을 독립 보관 |
| 소스 파일에 **읽기 권한이 없는** 사용자에게 제공 | **Deep** | shallow는 소스 파일 읽기 권한이 필요 |
| 파일이 **매우 많은** 테이블을 반복 복제 | **Shallow 우선 검토** | 이 측정에서 Deep는 파일 개수에 비례해 시간 증가 |
| **스트림을 클론에서 이어서** 실행 | **Deep** | 스트림 메타데이터는 Deep에서만 복제 |

# 8. 이 글의 근거 (무엇을 직접 측정했고, 무엇이 문서·추정인가)

| # | 주장 | 근거 유형 | 근거 내용 |
|---|---|---|---|
| 1 | 16개 테이블에서 Shallow가 평균적으로 더 빠르다 | **직접 테스트** | 16개 테이블 × 2방식 = 32회 측정, 14개 테이블에서 Shallow가 더 빠름 |
| 2 | 시간은 크기보다 파일 개수의 영향이 크다 (Deep에서 뚜렷) | **직접 테스트 + 직접 계산** | 이상치 제외 14개 기준 Deep R²(파일) 0.96 vs R²(크기) 0.03 |
| 3 | 4번 이상치는 첫 실행 지연일 수 있다 | **추정** | 실행 순서 미기록. 재측정 필요 |
| 4 | 이 크기에서 속도 차이는 몇 초 수준이다 | **직접 테스트** | 평균 차이 약 1~1.3초 |
| 5 | GB~TB에서는 Deep가 데이터 크기에 비례해 느려질 것이다 | **공식 문서(개념) / 직접 측정 없음** | 문서: Deep는 데이터를 복사해 비용이 큼. 이 규모는 미측정 |
| 6 | 비-UC에서는 소스 VACUUM 후 shallow가 깨지고 UC에서는 깨지지 않는다 | **공식 문서** | 직접 테스트 아님 |
| 7 | Dedicated 모드에서는 타깃 변경 시 소스 MODIFY가 필요하다 | **공식 문서 + 직접 테스트** | 소스에 `USE`·`SELECT`만 부여한 Dedicated 컴퓨트에서 최초 복제 후 작업이 `MODIFY on Table '<SOURCE>'`로 실패, Standard는 모두 성공 (별도 글 참고) |
| 8 | 증분 동기화는 CLONE 재실행으로 가능하다 | **공식 문서** | 직접 테스트 아님 |

- 측정 환경: Azure VM 시리즈(`Standard_D8ds_v5`) 클러스터. 공식 문서 링크는 **AWS** 문서이며 Azure 문서와의 차이는 확인하지 않았습니다.
- 이 사례는 포트폴리오 자료로도 정리했습니다. (`portfolio/` 폴더)

# 9. 체크리스트

- [ ] 대상 테이블이 **Unity Catalog 관리형/외부**인지, **Hive metastore**인지 확인했는가? (VACUUM 동작이 다름)
- [ ] Shallow를 쓸 때 **소스 파일 읽기 권한**과 **접근 모드(Standard/Dedicated)별 MODIFY 권한**을 확인했는가?
- [ ] 보관·DR 목적이면 **Deep**이고, **증분 동기화는 CLONE 재실행(스케줄)** 으로 구성했는가?
- [ ] **복제되지 않는 것**(이력, 태그, 설명)을 대상 테이블에서 별도로 처리해야 하는가?
- [ ] 측정할 때 **반복 횟수, 실행 순서, 워밍업, 런타임 버전**을 기록했는가?

# 10. 한 줄 정리

- **Shallow = 메타데이터만 복사**(빠르고 저렴, 소스 파일 의존), **Deep = 데이터까지 복사**(독립, 보관·DR용).
- 직접 측정(16개 테이블, 50~450MB)에서 Shallow가 대체로 더 빨랐고, **Deep 시간은 크기보다 파일 개수에 비례**했다(R² 0.96 vs 0.03). 단 **1회 측정, 이상치 2개, 소규모**라는 한계가 있다.
- 정리표에서 보강할 점: **UC에서는 소스 VACUUM이 shallow clone을 깨뜨리지 않고**, **증분 동기화는 CLONE 재실행**으로 가능하다.
- 속도보다 **독립성, VACUUM, 권한, 저장 공간**이 선택 기준이다.

# 관련 용어 글
Deep clone · Shallow clone · Delta 트랜잭션(ACID)

## 🔗 참고
- [Clone a table on Databricks](https://docs.databricks.com/aws/en/delta/clone)
- [Shallow clone for Unity Catalog tables](https://docs.databricks.com/aws/en/delta/clone-unity-catalog)
- [CREATE TABLE CLONE](https://docs.databricks.com/aws/en/sql/language-manual/delta-clone)
