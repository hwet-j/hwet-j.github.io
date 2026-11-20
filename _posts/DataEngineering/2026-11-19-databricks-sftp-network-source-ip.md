---
title:  "[DE/Databricks] 사내망 파일 서버(SFTP) 연동: Databricks 접속 Source IP와 경로 확인하기"
excerpt: "'Databricks의 Source IP가 사설인가 공인인가?'라는 질문에는 컴퓨트 종류와 네트워크 구성에 따라 답이 다릅니다"

categories:
  - DataEngineering
tags:
  - [Databricks, SFTP, Network, Classic Compute, Private Link]

toc: true
toc_sticky: true
published: true

date: 2026-11-19
last_modified_at: 2026-11-19
---

업무에서 겪은 상황을 일반화해서 정리했습니다. 이름·대역은 모두 **예시/가칭**이며 실제 값이 아닙니다.    
공식 문서(AWS 기준)를 참고하여 정리하였습니다. (확인일: 2026-11-19) 이후 **Classic 컴퓨트 + AWS**로 확정되어 7장에 구체 점검을 추가했습니다. Azure/GCP는 구성 방식이 다르며 확인하지 못했습니다.    
[SFTP 수집](https://docs.databricks.com/aws/en/ingestion/sftp) ·
[Classic 네트워킹](https://docs.databricks.com/aws/en/security/network/classic/) ·
[Serverless 네트워킹](https://docs.databricks.com/aws/en/security/network/serverless-network-security/) ·
[Serverless 방화벽](https://docs.databricks.com/aws/en/security/network/serverless-network-security/serverless-firewall) ·
[VPC 리소스 Private Link](https://docs.databricks.com/aws/en/security/network/serverless-network-security/pl-to-internal-network) ·
[SSH reverse tunnel](https://docs.databricks.com/aws/en/security/network/serverless-network-security/ssh-reverse-tunnel) ·
[VPC peering](https://docs.databricks.com/aws/en/security/network/classic/vpc-peering) ·
[Customer-managed VPC](https://docs.databricks.com/aws/en/security/network/classic/customer-managed-vpc)
{: .notice--warning}

# 1. 상황

한쪽 조직(Source)의 **파일 서버**에 있는 파일을, 다른 조직(Target)의 **Databricks**가 SFTP로 가져오려는 구성입니다. Source 쪽 **외부망 인바운드는 차단될 예정**이라서, 두 조직 사이에 **임시 전용 구간(임시 메트로)** 을 두기로 안내받았습니다.

```
 [인터넷]
    │
 [Target-Source F/W]──[Target_Net]            ← Target 내부망 (파란색)
    │
 [Source_B/B] ───── [Source L4_S/W]
    │
 [Source F/W]
    │
 [Source User B/B]
    │
 [Source 파일 서버]  (클라이언트 VM 60대 할당)
```

안내받은 데이터 흐름은 **Target_Net → 임시 메트로 → Source 측 파일 서버**입니다. 이때 Databricks가 **공인망(Public IP)** 으로 SFTP에 접속하면, Source의 인바운드 차단 정책에 막힐 수 있다는 점이 걱정입니다.

# 2. 확인하고 싶은 것 두 가지

| # | 질문 | 왜 중요한가 |
|---|---|---|
| ① | SFTP 접속에 쓰는 **Source IP**가 Target_Net의 **사설 IP**인가, **공인 IP**인가? | Source 쪽 방화벽에 **어떤 IP/대역을 허용 요청**해야 하는지 결정 |
| ② | Databricks가 공인 인터넷을 거치지 않고 **Private Link나 전용선**으로 Target_Net과 비공개로 연동되는가? | 인바운드 차단 환경에서 **접속 경로가 성립하는지** 결정 |

# 3. 핵심 답: Databricks의 Source IP는 "하나"가 아니다

Databricks 자체가 정해진 하나의 IP를 갖는 것이 아니라, **어떤 컴퓨트에서 접속하느냐**와 **고객이 구성한 네트워크 경로**에 따라 달라집니다. (→ 컴퓨트 플레인 용어 글)

## 3-1. Classic 컴퓨트 (내 VPC에서 실행)

- 클러스터는 **내 클라우드 계정의 VPC**에서 실행되고, 새 워크스페이스는 secure cluster connectivity가 기본이라 **노드에 공인 IP가 없습니다.**
- 외부로 나가는 경로는 **내가 구성한 라우팅**에 의해 결정됩니다.
- SFTP 공식 문서도 다음처럼 안내합니다.
  - 서버의 SSH 규칙이 **Databricks VPC의 IP 대역**을 허용해야 한다.
  - 고정 IP가 필요하면 **로드 밸런서·NAT 게이트웨이·인터넷 게이트웨이 등**을 컴퓨트 서브넷에 연결해 **허용 목록에 올릴 수 있는 안정적인 공인 IP**를 공유하게 한다.
- 즉 Classic의 Source IP는
  - **NAT/인터넷 게이트웨이 경유** → 해당 장치의 **공인 IP**
  - **peering·VPN·전용선 등 사설 경로 경유** → **VPC의 사설 IP 대역** (또는 경로 중간 장비에서 NAT된 IP)

## 3-2. Serverless 컴퓨트 (Databricks 관리 영역에서 실행)

- 같은 리전의 S3·DynamoDB 이외의 리소스에는 **공인 IP**로 접속합니다.
- 이 IP는 Databricks가 **`ip-ranges.json`으로 게시**하고, **주기적으로 바뀔 수 있어** 허용 목록 갱신을 자동화해야 합니다. (→ 서버리스 아웃바운드 IP 글)
- 사설 연결을 원하면 **NCC의 private endpoint**(VPC의 NLB 뒤 리소스 대상)를 사용합니다. (→ NCC 글)

## 3-3. 요약표

| | Classic | Serverless |
|---|---|---|
| 접속 Source IP | **내 라우팅 구성에 따라** (NAT의 공인 IP 또는 사설 IP) | Databricks가 게시한 **공인 아웃바운드 IP** |
| 고정 가능? | NAT/LB로 **고정 가능** | 게시 대역, **변경될 수 있음** |
| 사설 연결 | 내 VPC 라우팅(peering, VPN, 전용선) | **NCC + Private Link** 등 별도 구성 |
| SFTP 서버 허용 대상 | VPC 대역 또는 NAT 공인 IP | 게시된 아웃바운드 IP |

# 4. 질문 ②의 함정: "Private Link"는 두 가지가 있다

문의 문안의 "Private Link"는 의미를 구분해야 합니다.

| 구분 | 용도 | 이번 SFTP 접속과의 관계 |
|---|---|---|
| (a) **Databricks 자체 연결용** Private Link | 사용자↔워크스페이스(front-end), **컨트롤 플레인↔컴퓨트 플레인**(back-end) | **무관**. 파일 서버로 가는 데이터 트래픽 경로를 만들지 않음 |
| (b) **서버리스 outbound Private Link** (NCC) | 서버리스 → **내 VPC의 NLB 뒤 리소스** | 대상이 내 VPC 안에 있을 때 사용. 외부 사내망은 VPC까지의 경로가 따로 필요 |

즉 **"Databricks가 기본적으로 사설망으로 Target_Net과 연결된다"는 보장은 없으며**, 사설 경로는 **고객이 직접 구성**해야 합니다. Classic 컴퓨트에서 온프레미스/타 조직 사설망으로 가는 구체적 라우팅(전용선, VPN, Transit 등)을 Databricks 문서가 자세히 안내하는 페이지는 이번에 확인하지 못했습니다. 일반 클라우드 네트워킹으로 구성하는 영역이라 **클라우드 네트워크 담당과 확인**이 필요합니다.

# 5. 구성도에 대입해 보기: 경로 시나리오 5가지

| 시나리오 | 경로 | Source 쪽 방화벽이 보게 될 IP | Source 인바운드 차단 환경에서 |
|---|---|---|---|
| **A. Classic + 사설 경로** | Databricks VPC → (전용선/VPN) → Target_Net → 임시 메트로 → Source | **Target_Net 대역의 사설 IP** (또는 Target_Net 경계에서 NAT된 IP) | ✅ 성립 가능 (사설 경로가 열려 있다면) |
| **B. Classic + NAT 공인망** | Databricks VPC → NAT → 인터넷 → Source | NAT의 **공인 IP** | ❌ 인바운드 차단 시 막힘 |
| **C. Serverless 직접** | Serverless → 인터넷 → Source | Databricks 게시 **공인 아웃바운드 IP** | ❌ 인바운드 차단 시 막힘 |
| **D. Serverless + 사설 연결** | Serverless → NCC private endpoint → (VPC의 NLB) → (전용선/VPN) → 사설망 → Source | 경로 구성에 따름 | ✅ 가능하나 구성 복잡(NLB 등) |
| **E. 사내 → 아웃바운드 터널** | 사내 터널 호스트가 프록시 VM으로 **먼저 SSH 연결**, Databricks는 프록시 경유 | 프록시 구성에 따름 | ✅ 인바운드 불필요, **직접 운영 부담** (→ SSH reverse tunnel 글) |

**중요한 점**: Source 쪽 방화벽이 실제로 보게 되는 IP는 **Databricks의 IP 그 자체가 아니라, Target_Net 및 임시 메트로 구간에서 NAT(주소 변환)가 일어난 뒤의 IP**일 수 있습니다. 그래서 ①번 질문은 Databricks 담당뿐 아니라 **Target_Net → 임시 메트로 구간의 NAT 여부를 아는 네트워크 담당**도 함께 답해야 합니다.

# 6. 우리 쪽에서 직접 확인하는 방법

(아래 명령은 일반적인 확인 방법이며, Databricks 공식 문서의 절차가 아닙니다.)

## 6-1. 컴퓨트 종류부터 확인

- 어떤 컴퓨트로 SFTP 작업이 돌 예정인가? (Classic 클러스터 / Serverless / Lakeflow 파이프라인) — SFTP 커넥터는 **모든 컴퓨트 유형, DBR 17.3 이상**에서 사용 가능합니다.

## 6-2. Classic이라면

1. **워크스페이스 VPC 서브넷의 라우트 테이블** 확인
   - `0.0.0.0/0`이 **NAT 게이트웨이/인터넷 게이트웨이**로 가는지
   - **Target_Net 대역(CIDR)** 이 **전용선/VPN/Transit/peering**으로 향하는 라우트가 있는지
2. 클러스터 노드의 **사설 IP 대역**을 확인 (SFTP 서버가 허용해야 할 후보 대역)
3. 노트북에서 도달성 테스트

```
%sh nc -zv <sftp-host-or-ip> 22        # 포트 22 연결 가능 여부
%sh ip route                           # 노드에서 보이는 라우팅
```

4. 경로 중간에 **NAT 장비**가 있는지 네트워크 담당에게 확인

## 6-3. Serverless라면

- 허용해야 할 IP는 **`ip-ranges.json`** 에서 (`service=Databricks`, `type=outbound`, 내 리전·플랫폼) 필터한 **`ipv4Prefixes`** 입니다.
- 인바운드가 막힌 사설 서버에는 **공인 IP 허용 방식이 맞지 않으므로** 사설 연결(NCC, 터널)을 검토해야 합니다.

## 6-4. SFTP 연결 테스트

- Unity Catalog connection 생성 화면의 **Test and create connection**에서 연결을 시험하고, **호스트 키 지문**도 오류 메시지로 확인할 수 있습니다. (→ SFTP 커넥터 글)

# 7. 확정 조건: Classic 컴퓨트 + AWS 에서의 구체 점검

## 7-1. 공식 문서가 말하는 전제 (customer-managed VPC 기준)

- 워크스페이스 서브넷은 **NAT 게이트웨이 + 인터넷 게이트웨이**, 또는 **고객이 관리하는 유사 장비**를 통해 공용망으로 나가는 **outbound 경로**가 있어야 합니다.
- 서브넷 **라우트 테이블의 `0.0.0.0/0`** 은 **NAT 게이트웨이 또는 직접 관리하는 NAT 장비/프록시 어플라이언스**를 가리켜야 합니다. (Databricks 컨트롤 플레인 등과의 통신을 위한 요건)
- **서브넷 네트워크 ACL**은 어떤 트래픽도 ingress/egress에서 **거부하면 안 되고**, 워크스페이스 **VPC CIDR 내부 트래픽을 모두 허용**해야 합니다.
- **외부 방화벽**(AWS Network Firewall 또는 Gateway Load Balancer 연동 서드파티)은 classic 컴퓨트 egress 검사용으로 **선택(격리 환경에서는 필수)** 입니다. 이 경우 **클러스터와 Databricks 컨트롤 플레인 사이의 TLS 검사(복호화/재암호화)는 하지 말라**고 합니다. 클러스터 장애의 원인이 됩니다.
- SFTP 문서의 전제: **원격 서버가 워크스페이스 VPC에서 접근 가능**해야 하고, 서버의 SSH 규칙이 **Databricks VPC의 IP 대역**을 허용해야 합니다.
- 사설 연결을 위한 VPC peering은 상대 VPC의 CIDR이 **Databricks VPC 및 라우트 테이블 목적지의 다른 CIDR과 겹치지 않아야** 합니다.

## 7-2. 경로를 가르는 핵심: 서브넷 라우트 테이블

(이 절은 일반적인 AWS 라우팅 원리이며 Databricks 공식 문서의 절차가 아닙니다. 구성 전 AWS 네트워크 담당과 확인하세요.)

```
워크스페이스 서브넷 라우트 테이블
 ├─ 0.0.0.0/0        → NAT 게이트웨이            (Databricks 필수. 인터넷/기본 경로)
 └─ <Target_Net CIDR> → 전용선/VPN용 게이트웨이     (예: Transit Gateway, Virtual Private Gateway)
                         ← 이 라우트가 있어야 SFTP 트래픽이 사설 경로로 감
```

- 더 구체적인(긴 접두사) 라우트가 우선하므로, **Target_Net 대역으로 가는 라우트가 있으면** SFTP 서버행 트래픽은 사설 경로로, 나머지는 `0.0.0.0/0`(NAT)로 갑니다.
- **그 라우트가 없으면** SFTP 트래픽도 `0.0.0.0/0` → NAT → **공인 IP**로 나가고, Source 인바운드 차단 시 막힙니다. (앞의 시나리오 B)

| 라우트 구성 | Source 방화벽이 보는 IP | 인바운드 차단 환경 |
|---|---|---|
| Target_Net 라우트 **있음** (전용선/VPN/Transit) | 클러스터 노드의 **사설 IP**(워크스페이스 서브넷 CIDR 안) 또는 경로 중간에서 NAT된 IP | ✅ 가능 |
| Target_Net 라우트 **없음** (`0.0.0.0/0`만) | **NAT 게이트웨이의 공인 IP** | ❌ 막힘 |
| 외부 방화벽/프록시가 **경로에 있음** | 방화벽/프록시의 **IP**(장비에서 NAT되면 그 IP) | 구성에 따라 |

## 7-3. Source 쪽에 어떤 IP를 허용 요청해야 하나 (사설 경로일 때)

- 클러스터 노드는 **워크스페이스 서브넷 CIDR 안의 사설 IP**를 쓰고, **노드가 바뀔 때마다 그 안에서 달라질 수 있습니다.** 그래서 보통은 **서브넷 CIDR 전체**(SFTP 문서의 "Databricks VPC IP 대역")를 허용 요청합니다.
- **단일 IP**만 허용하고 싶다면 선택지는 다음과 같습니다.
  - **Target_Net 경계(임시 메트로 입구)에서 하나의 IP로 NAT(SNAT)** 하도록 요청
  - SFTP 용도로만 쓰는 **작은 전용 서브넷/전용 클러스터**를 만들어 허용 범위를 최소화
- **CIDR 겹침 확인**: 워크스페이스 VPC CIDR이 **Target_Net/Source 내부 대역과 겹치면** 라우팅이 성립하지 않습니다. 겹치면 대역 조정이나 NAT가 필요합니다.
- 서버를 **호스트명**으로 접속한다면 **DNS가 사설 IP로 해석되는지**도 확인하세요. (공인 DNS가 공인 IP를 돌려주면 경로가 어긋납니다)

## 7-4. 이 조건에서 바로 해볼 확인 순서

1. 워크스페이스 VPC ID와 **클러스터가 배치된 서브넷**, 그 서브넷의 **CIDR**
2. 서브넷 **라우트 테이블**에서 `0.0.0.0/0`의 대상(NAT?)과 **Target_Net CIDR 라우트 유무**
3. 서브넷 CIDR과 Target_Net/Source 대역의 **겹침 여부**
4. 경로에 **외부 방화벽/프록시/NAT 장비**가 있는지 (Target_Net 경계 포함)
5. Classic 클러스터 노트북에서 도달성 테스트

```
%sh nc -zv <sftp-host-or-private-ip> 22
%sh ip route
```

6. Unity Catalog에서 SFTP connection **Test and create connection**으로 최종 확인

# 8. 문의 문안 개선 제안

원래 문안은 방향이 좋지만, **상대가 정확히 답할 수 있도록** 정보를 더 구체화하면 왕복이 줄어듭니다.

> **[배경]** Databricks에서 Source 측 파일 서버로 SFTP 접속 시, Source 외부망 인바운드가 차단될 예정이어서 임시 메트로를 통한 사내망 연동 방식이 맞는지 확인이 필요합니다.
>
> **[Databricks 측 확인 요청]**
> 1. SFTP 접속에 사용할 **컴퓨트 종류**(Classic / Serverless)와 클라우드·리전
> 2. 접속 시 **Source IP**: Target_Net 대역의 **사설 IP**인지, **공인 IP**(NAT 등)인지. 사설이라면 **어떤 대역(CIDR)** 인지
> 3. Databricks → Target_Net 사이 **연결 방식**(전용선 / VPN / peering / Transit 등)과, 공인 인터넷을 **경유하지 않는지**
>
> **[Target 네트워크 측 확인 요청]**
> 4. Target_Net → 임시 메트로 구간에서 **NAT(SNAT) 적용 여부**, 적용 시 Source가 보게 될 **IP/대역**
>
> **[Source 방화벽 허용 요청 정보(회신 후 확정)]**
> 5. 허용할 **Source IP/대역, 목적지 IP, 포트(SFTP 22), 프로토콜(TCP)**, 변경 가능성

**Classic + AWS로 확정된 경우의 간소화 문안**

> 1. Databricks 워크스페이스(Classic, AWS)의 **클러스터 서브넷 CIDR**과 라우트 테이블에서 **Target_Net 대역이 전용선/VPN(Transit 등)으로 라우팅되는지** 확인 부탁드립니다. (`0.0.0.0/0`은 NAT 게이트웨이로 향함)
> 2. 따라서 SFTP 접속 Source IP는 **클러스터 서브넷의 사설 IP 대역**인지, 중간 장비에서 **NAT된 IP**인지 확인 부탁드립니다.
> 3. Target_Net → 임시 메트로 구간의 **NAT(SNAT) 여부**와, Source가 보게 될 **단일 IP/대역**을 회신 부탁드립니다. (단일 IP로 고정이 필요하면 SNAT 적용 가능 여부)
> 4. 워크스페이스 VPC CIDR과 Target_Net/Source 대역의 **겹침 여부** 확인 부탁드립니다.

# 9. 체크리스트

- [ ] SFTP를 **어떤 컴퓨트**(Classic/Serverless)로 실행하는가?
- [ ] Source 서버가 보는 IP는 **Databricks IP가 아니라 NAT 이후의 IP**일 수 있음을 확인했는가?
- [ ] 접속 경로가 **공인 인터넷을 거치지 않는가**? (전용선/VPN/peering/Private Link)
- [ ] **Serverless라면 IP가 바뀔 수 있다**는 점과 갱신 절차를 정했는가?
- [ ] 서버 계정은 **최소 권한**(읽기 전용), 인증은 **PEM 키**, **호스트 키 지문 검증**을 켰는가?
- [ ] Unity Catalog connection의 비밀 값은 **secret scope**로 관리하는가?

# 10. 한 줄 정리

- Databricks의 **Source IP는 컴퓨트 종류와 내가 구성한 네트워크에 따라 달라진다.** Classic은 **내 라우팅/NAT**, Serverless는 **게시된 공인 아웃바운드 IP**.
- **"Private Link가 있으니 사설로 간다"는 보장이 없다.** Databricks 자체 연결용 Private Link와 SFTP 접속 경로는 다르다. 사설 경로는 **직접 구성**한다.
- 인바운드 차단 환경에서는 **사설 경로가 있는 Classic**, **NCC Private Link**, 또는 **SSH reverse tunnel** 중 하나가 필요하다.
- 문의 시에는 **컴퓨트 종류, 경로, NAT 여부, 허용할 IP/포트**를 구체적으로 묻는다.

# 관련 용어 글
컴퓨트 플레인(Classic vs Serverless) · NCC · 서버리스 아웃바운드 IP · SSH reverse tunnel · Databricks SFTP 커넥터

## 🔗 참고
- [Ingest files from SFTP servers](https://docs.databricks.com/aws/en/ingestion/sftp)
- [Classic compute plane networking](https://docs.databricks.com/aws/en/security/network/classic/)
- [Serverless compute plane networking](https://docs.databricks.com/aws/en/security/network/serverless-network-security/)
- [Serverless compute firewall configuration](https://docs.databricks.com/aws/en/security/network/serverless-network-security/serverless-firewall)
- [Configure private connectivity to resources in your VPC](https://docs.databricks.com/aws/en/security/network/serverless-network-security/pl-to-internal-network)
- [Connect to on-premises resources using an SSH reverse tunnel](https://docs.databricks.com/aws/en/security/network/serverless-network-security/ssh-reverse-tunnel)
- [VPC peering](https://docs.databricks.com/aws/en/security/network/classic/vpc-peering)
- [Configure a customer-managed VPC](https://docs.databricks.com/aws/en/security/network/classic/customer-managed-vpc)
- [Isolated environment (egress firewall)](https://docs.databricks.com/aws/en/security/network/deployment-architecture/isolated-environment)
