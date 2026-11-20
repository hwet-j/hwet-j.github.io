---
title:  "[DE/용어] Databricks SQL UI cache"
excerpt: "SQL 편집기와 레거시 SQL 대시보드에서 최근 결과를 바로 보여주는 사용자별 캐시"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, Cache, SQL UI Cache, SQL Editor]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[Query caching](https://docs.databricks.com/aws/en/sql/user/queries/query-caching)
{: .notice--warning}

# 0. 이 글을 읽기 전에

SQL 편집기에서 쿼리를 열면 **방금 실행하지 않았는데도 지난 결과가 먼저 보이는** 경험이 있을 것입니다. 이것이 **Databricks SQL UI cache**입니다. 이름이 비슷한 **AI/BI 대시보드 캐시**와 다르니 구분해서 읽으세요.

# 1. 한 줄 정의

**사용자별**로 **쿼리 결과와 SQL 편집기 시각화**를 캐시하는 Databricks SQL의 UI 캐시입니다. 사용자가 **SQL 쿼리 또는 레거시 SQL 대시보드를 처음 열 때** 가장 최근 쿼리 결과(**예약 실행 결과 포함**)를 화면에 보여줍니다.

# 2. 핵심 사항 (공식 문서)

| 항목 | 내용 |
|---|---|
| 범위 | **사용자별(per-user)** |
| 대상 | **SQL 편집기**, **레거시 SQL 대시보드** |
| 수명 | **최대 7일** |
| 위치 | **내 계정의 Databricks 파일시스템** |
| 삭제 | 지우고 싶은 쿼리 결과는 **쿼리를 다시 실행**하면 옛 결과가 캐시에서 제거됨 |
| 무효화 | **기저 테이블이 갱신되면** 무효화 |
| 적용 제외 | **AI/BI 대시보드(구 Lakeview)에는 적용되지 않음** — 별도 캐시 동작 |

# 3. 다른 캐시와의 관계

```
SQL UI cache (사용자별 화면용 결과, ≤7일)
   └─ 화면에 보여줄 결과가 없거나 무효화되면 → 쿼리 실행
        └─ SQL 웨어하우스의 쿼리 결과 캐시 (local → remote)
             └─ 실행 시 Disk cache
```

- UI cache는 **화면 표시용 사용자별 결과**이고, 웨어하우스 쪽의 **쿼리 결과 캐시**(클러스터/워크스페이스 단위, 24시간)와는 **별개**입니다.
- 쿼리 결과 캐시와 Disk cache는 **Databricks SQL UI, BI 도구, 외부 클라이언트**의 쿼리에 모두 영향을 줍니다.

# 4. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "모든 Databricks 대시보드는 7일 캐시" | **7일은 SQL UI cache**(편집기·레거시 SQL 대시보드). **AI/BI 대시보드는 24시간 캐시** |
| "화면 렌더링만 돕는다" | **사용자별 쿼리 결과**를 보관 |
| "테이블이 바뀌어도 예전 값을 계속 보여준다" | **기저 테이블이 갱신되면 무효화** |
| "캐시를 지울 방법이 없다" | **쿼리를 다시 실행**하면 옛 결과가 제거됨 |

# 5. 한 줄 정리

- SQL UI cache = **SQL 편집기·레거시 SQL 대시보드를 열 때 최근 결과를 바로 보여주는 사용자별 캐시**. **최대 7일**, 테이블 갱신 시 무효화.
- **AI/BI 대시보드에는 적용되지 않는다.** (AI/BI 대시보드 캐시 글 참고)

# 관련 용어
SQL 쿼리 결과 캐시, AI/BI 대시보드 캐시

## 🔗 참고
- [Query caching](https://docs.databricks.com/aws/en/sql/user/queries/query-caching)
