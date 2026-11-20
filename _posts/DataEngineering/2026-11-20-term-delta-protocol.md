---
title:  "[DE/용어] Delta 프로토콜과 Table feature"
excerpt: "기능을 켤 때 '테이블 프로토콜이 올라간다'는 말의 의미와, 읽기/쓰기 호환성에 미치는 영향"

categories:
  - DataEngineering
tags:
  - [Glossary, Delta Lake, Protocol, Table Feature]

toc: true
toc_sticky: true
published: true

date: 2026-11-20
last_modified_at: 2026-11-20
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-20)    
[Delta Lake feature compatibility and protocols](https://docs.databricks.com/aws/en/delta/feature-compatibility)
{: .notice--warning}

# 0. 이 글을 읽기 전에

Liquid Clustering, Deletion vectors, Type widening 글마다 **"프로토콜이 업그레이드되어 일부 클라이언트가 읽지 못할 수 있다"** 는 경고가 나옵니다. 이 글은 그 말이 정확히 무슨 뜻인지 설명합니다.

# 1. 한 줄 정의

모든 Delta 테이블에는 **프로토콜(protocol)** 이라는 사양이 있어서, 그 테이블을 **읽거나 쓰려면 클라이언트가 어떤 기능들을 지원해야 하는지**를 정해 둡니다. 어떤 클라이언트가 테이블이 쓰는 기능을 지원하지 못하면 **그 테이블을 읽거나 쓸 수 없습니다.**

# 2. 쉬운 비유 (이해를 돕기 위한 비유)

테이블에 **"이 문서를 열려면 최소 이 버전의 프로그램이 필요합니다"** 라는 문구가 붙어 있다고 생각해 보세요. 새로운 기능(예: Deletion vectors)을 켜면 문구가 **"더 높은 버전 필요"** 로 바뀝니다. 낮은 버전의 프로그램(오래된 Databricks Runtime이나 외부 도구)은 그 문서를 열지 못합니다.

# 3. 용어 정리 (공식 문서)

| 용어 | 설명 |
|---|---|
| **Delta Lake client** | Delta 테이블을 읽거나 쓰는 모든 시스템 |
| **Read protocol** | 테이블을 **읽기 위해** 클라이언트가 지원해야 할 것 |
| **Write protocol** | 테이블에 **쓰기 위해** 클라이언트가 지원해야 할 것 |
| `minReaderVersion` | 읽기 프로토콜의 정수 값 (유효: 1, 2, 3) |
| `minWriterVersion` | 쓰기 프로토콜의 정수 값 (유효: 2~7) |
| **Table feature** | 프로토콜 버전보다 **세밀한 단위**의 기능 플래그 |

- **쓰기 프로토콜/쓰기 기능**은 **쓰는 클라이언트의 호환성에만** 영향을 주므로, 오래된 읽기 전용 워크로드는 계속 접근할 수 있습니다.
- 모든 Delta 기능이 서로 호환되는 것은 아닙니다.

# 4. 프로토콜 버전 vs Table feature

- **예전 방식**: 정수 버전(`minReaderVersion`, `minWriterVersion`)에 여러 기능이 묶여 있고 **누적**됩니다.
- **현재 방식**: **DBR 12.2 LTS 이상**에서는 **table feature**가 정수 프로토콜을 대체합니다. 테이블이 어떤 기능을 쓰는지를 **세밀한 플래그**로 나타내 더 정밀한 호환성 검사가 가능합니다. (DBR 11.3 LTS 이하에도 이미 지원하던 기능에 한해 table feature가 백포트됨)

| 구분 | 영향 | 요구 버전 |
|---|---|---|
| **Writer feature** | 데이터를 **쓰는 방식**에 영향. 읽기 클라이언트는 막지 않음 | `minWriterVersion = 7` |
| **Reader feature** | 데이터를 **읽는 방식**에 영향. 모든 reader feature는 writer feature이기도 함 | `minReaderVersion = 3`, `minWriterVersion = 7` |

- "읽을 수 없는 테이블에는 쓸 수도 없다": 클라이언트는 **읽지 못하는 테이블에 쓸 수 없습니다.**

## 기능별 요구 버전 (일부 발췌, 공식 문서 표)

| 기능 | minWriterVersion | minReaderVersion | 유형 |
|---|---|---|---|
| CHECK 제약 | 3 | 1 | 쓰기만 |
| Change data feed | 4 | 1 | 쓰기만 |
| Generated columns | 4 | 1 | 쓰기만 |
| Column mapping | 5 | 2 | 읽기+쓰기 |
| Identity columns | 6 | 1 | 쓰기만 |
| Row tracking | 7 | 1 | 쓰기만 |
| **Deletion vectors** | 7 | 3 | **읽기+쓰기** |
| TimestampNTZ | 7 | 3 | 읽기+쓰기 |

→ **Deletion vectors**는 reader feature라서 이를 모르는 오래된 클라이언트는 **읽지도 못합니다.** 반면 CHECK 제약은 writer feature라서 **읽기 전용 클라이언트는 계속 읽을 수 있습니다.**

# 5. 프로토콜은 언제 올라가는가

**새 기능을 켜면 프로토콜이 자동으로 업그레이드**됩니다. 트리거는 두 가지입니다.

1. **CREATE/ALTER 문법**으로 자동 활성화
   - `CREATE TABLE ... CLUSTER BY (...)` → **Liquid clustering** 활성화
   - `GENERATED ALWAYS AS` → **Generated columns** 활성화
2. **테이블 속성으로 명시적 활성화**
   - `'delta.enableDeletionVectors' = true` → **Deletion vectors**
   - `'delta.enableTypeWidening' = 'true'` → **Type widening**

- 어떤 기능을 켜면 **필요한 다른 기능이 함께 켜질 수** 있습니다. 예: Iceberg 읽기를 켜면 column mapping이, liquid clustering을 켜면 checkpoint V2가 자동으로 켜집니다.
- **Reader feature**는 읽기·쓰기 프로토콜을 **모두** 올리고(데이터 저장 방식이 달라지므로), **Writer feature**(예: CHECK 제약)는 **쓰기 프로토콜만** 올립니다.

> ⚠️ 공식 경고: **대부분의 프로토콜 업그레이드는 되돌릴 수 없고**, 기존 Delta 리더/라이터를 **깨뜨릴 수 있습니다.** 필요한 테이블에만 업그레이드하고, 현재와 앞으로의 프로덕션 도구가 새 프로토콜을 지원하는지 먼저 확인하세요.
{: .notice--warning}

# 6. 되돌리기(다운그레이드)

- 프로토콜 변경 규칙: **기능을 켜면 업그레이드, 기능을 제거(drop)하면 다운그레이드**됩니다.
- **기능을 "끄는 것"(속성을 false로)만으로는 프로토콜이 내려가지 않습니다.** 완전히 제거하려면 `DROP FEATURE`가 필요합니다. 모든 기능을 제거할 수 있는 것은 아닙니다.
- Databricks는 **`minReaderVersion`/`minWriterVersion` 속성을 직접 바꾸지 말라**고 권장합니다. 값을 낮춰도 다운그레이드되지 않고, 업그레이드를 막지도 못합니다.
- 프로토콜 변경은 **동시 쓰기와 충돌**하고, 테이블 메타데이터를 바꾸는 커밋을 만난 **스트리밍 읽기는 실패**합니다(재시작 필요).
- Delta는 프로토콜을 **활성 기능을 모두 지원하는 가장 낮은 버전**으로 해소합니다. 낮은 정수 버전에서 모두 지원되면 `readerFeatures`/`writerFeatures`가 사라질 수 있지만, **기능이 꺼지는 것은 아닙니다.**

# 7. Databricks Runtime과 외부 시스템 호환성

- 새 Delta 기능은 **Databricks Runtime 릴리스**에서 지원이 시작됩니다.
- **낮은 DBR로 쓴 테이블**은 높은 DBR에서 **완전히 읽고 쓸 수 있습니다.** 반대로 **높은 DBR로 쓴 테이블**은 낮은 DBR이 **지원하지 않는 기능**을 쓸 수 있습니다.
- Databricks 안에서만 쓴다면 **최소 DBR 버전**만 관리하면 됩니다. **외부 시스템**에서 테이블을 읽거나 쓴다면, 그 클라이언트가 **활성화된 table feature를 지원하는지 반드시 확인**해야 합니다.
- Lakeflow 파이프라인과 Databricks SQL은 정기 릴리스로 런타임이 **자동 업그레이드**됩니다.
- OSS Delta Lake 클라이언트는 기능별 지원을 **스스로 구현**하므로, 프로덕션 테이블에서 기능을 켜기 전에 **호환성 테스트**를 하라고 안내합니다.

# 8. 이 시리즈 글과의 연결

| 기능 | 프로토콜 영향 |
|---|---|
| **Liquid clustering** | Delta writer **7** / reader **3**. 지원하지 않는 클라이언트는 **읽지 못하고** 다운그레이드 불가 |
| **Deletion vectors** | 켜면 프로토콜 업그레이드. 미지원 클라이언트는 **읽을 수 없음**. 지원하는 DBR에서 기능을 drop해 호환시킬 수 있음 |
| **Type widening** | 켜면 reader/writer 프로토콜 업그레이드. **DBR 15.4 LTS 이상만** 읽을 수 있음 |

# 9. 확인과 점검

- 테이블의 프로토콜/기능 정보는 `DESCRIBE DETAIL <table>`로 확인할 수 있습니다. (표시되는 컬럼의 세부는 공식 문서의 DESCRIBE DETAIL 참고)
- 기능을 켜기 전 체크리스트
  - [ ] 이 테이블을 **읽는 모든 시스템**(BI 도구, 다른 클러스터, 외부 엔진)을 파악했는가?
  - [ ] 그 시스템들이 **해당 table feature를 지원**하는가? (DBR 버전 포함)
  - [ ] **되돌릴 수 있는 기능**인가? (`DROP FEATURE` 지원 여부)
  - [ ] 변경이 **동시 쓰기·스트림**에 주는 영향(충돌, 재시작)을 조율했는가?

# 10. 자주 하는 오해

| 오해 | 실제 |
|---|---|
| "기능을 false로 끄면 프로토콜이 원복된다" | **원복되지 않음.** `DROP FEATURE`로 제거해야 함 |
| "프로토콜 버전 속성을 낮추면 호환된다" | 속성을 바꿔도 다운그레이드되지 않음. 직접 수정 **비권장** |
| "읽기 전용 도구는 항상 문제없다" | **reader feature**가 켜지면 읽기도 불가 |
| "Databricks에서 잘 되니 다른 도구도 된다" | 외부 클라이언트는 table feature 지원을 **별도 확인** |

# 11. 한 줄 정리

- 프로토콜 = **테이블을 읽고 쓰려면 클라이언트가 지원해야 하는 기능 목록**. 현재는 정수 버전보다 **table feature**(DBR 12.2 LTS+)로 표현한다.
- **새 기능을 켜면 프로토콜이 올라가고**, 대부분 **되돌릴 수 없다.** 끄는 것(false)으로는 안 내려가고 `DROP FEATURE`가 필요하다.
- **Reader feature**(예: Deletion vectors, Column mapping)는 **읽기까지 막고**, **Writer feature**(예: CHECK 제약)는 **쓰기만** 제한한다.
- 외부 시스템이 읽는 테이블은 **기능을 켜기 전에 호환성을 확인**한다.

# 관련 용어
Liquid Clustering, Deletion vectors, Type widening, 스키마 진화(Schema evolution)

## 🔗 참고
- [Delta Lake feature compatibility and protocols](https://docs.databricks.com/aws/en/delta/feature-compatibility)
