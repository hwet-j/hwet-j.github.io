---
title:  "[DE/용어] 컴퓨트 플레인: Classic vs Serverless"
excerpt: "Databricks의 코드가 실제로 '어디서' 실행되는지, 그리고 그것이 접속 Source IP를 어떻게 바꾸는지"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, Network, Classic Compute, Serverless Compute]

toc: true
toc_sticky: true
published: true

date: 2026-11-19
last_modified_at: 2026-11-19
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-19)    
[Classic compute plane networking](https://docs.databricks.com/aws/en/security/network/classic/)    
[Serverless compute plane networking](https://docs.databricks.com/aws/en/security/network/serverless-network-security/)    
[VPC peering](https://docs.databricks.com/aws/en/security/network/classic/vpc-peering)
{: .notice--warning}

# 0. 이 글을 읽기 전에

"Databricks에서 외부 서버에 접속할 때 Source IP가 뭐죠?"라는 질문에는 **"어떤 컴퓨트로 접속하느냐에 따라 다릅니다"** 가 답입니다. 그 이유를 이해하려면 Databricks가 **어디서 코드를 실행하는지**부터 알아야 합니다. 이 글의 공식 문서 링크는 **AWS 기준**입니다. Azure·GCP는 서비스 이름과 구성 방식이 다르며 이번에 모두 확인하지는 못했습니다.

# 1. 한 줄 정의

Databricks는 크게 **컨트롤 플레인(control plane)** 과 **컴퓨트 플레인(compute plane)** 으로 나뉩니다. 컴퓨트 플레인은 **내 코드(노트북, Job, SQL)가 실제로 실행되는 곳**이고, 두 종류가 있습니다.

| | **Classic compute plane** | **Serverless compute plane** |
|---|---|---|
| 실행 위치 | **내(고객) 클라우드 계정의 VPC** | **Databricks가 관리하는 계정/네트워크** |
| 네트워크 주도권 | 고객이 VPC·서브넷·라우팅·NAT를 구성 | Databricks가 관리, 고객은 NCC·방화벽 허용 등으로 제어 |
| 노드의 공인 IP | 없음(secure cluster connectivity 기본) | 없음 (공식 문서: serverless 리소스에 공인 IP 없음) |

# 2. 구조 개념도

```
        ┌───────────────────────────┐
        │  Control plane (Databricks) │   웹 UI, REST API, 작업 스케줄 등
        └────────────┬──────────────┘
                     │ (암호화된 연결)
     ┌───────────────┴────────────────┐
     │                                │
┌────▼──────────────┐      ┌──────────▼───────────────┐
│ Classic compute    │      │ Serverless compute       │
│ (내 VPC 안에서 실행) │      │ (Databricks 관리 영역)     │
│ 라우팅·NAT: 내가 구성 │      │ 접속은 아웃바운드 IP /     │
└────┬──────────────┘      │ Private Link(NCC) 로 제어  │
     │                      └──────────┬───────────────┘
     ▼                                 ▼
 내 네트워크/외부 서버(예: SFTP 서버)      내 리소스/외부 서버
```

공식 문서는 컨트롤 플레인과 서버리스 컴퓨트 플레인 사이의 연결이 **항상 클라우드 네트워크 백본을 통하며 공용 인터넷을 거치지 않는다**고 설명합니다.

# 3. Classic compute 자세히

## 3-1. 내 VPC 안에서 실행된다

- Classic compute의 클러스터는 **내 클라우드 계정의 VPC**에서 실행됩니다. 기본적으로 Databricks가 VPC를 만들어 주지만, **직접 VPC를 지정(customer-managed VPC)** 해서 네트워크를 더 통제할 수도 있습니다. (워크스페이스 생성 시에 지정해야 함)
- 그래서 **외부로 나가는 경로(라우팅, NAT)는 내가 정한 네트워크 설계에 따라** 결정됩니다.

## 3-2. Secure cluster connectivity (SCC)

새 워크스페이스는 기본적으로 **secure cluster connectivity**를 사용합니다.

- 고객 VPC에 **열린 인바운드 포트가 없고**, classic compute 리소스에 **공인 IP가 없습니다.**
- 클러스터는 컨트롤 플레인과 **HTTPS(443) 보안 터널**로 연결합니다(SCC relay 사용).
- 즉 "클러스터 노드가 공인 IP를 갖고 인터넷에 나간다"는 구조가 **아닙니다.** 외부로 나갈 때는 보통 **NAT 같은 장치를 거쳐** 나가고, 이때 외부에서 보이는 IP는 **그 NAT 장치의 IP**입니다.

## 3-3. 다른 사설 네트워크와 연결하기

- 같은 클라우드의 다른 VPC와는 **VPC peering**으로 **사설 IP로** 통신할 수 있습니다. 공식 문서: "VPC peering은 Databricks 클러스터가 RDS, Redshift, Kafka 등 다른 인프라에 **내부 네트워크의 사설 IP**로 연결하게 해 준다."
- 조건: 연결할 VPC의 CIDR 대역이 **Databricks VPC 및 라우트 테이블 목적지의 다른 CIDR과 겹치지 않아야** 합니다. 겹치면 Databricks 지원 팀에 문의해 CIDR 이전을 요청해야 할 수 있습니다.
- 온프레미스 같은 외부 사설망으로 가는 경로(전용선, VPN 등)를 Databricks 문서가 자세히 안내하는 페이지는 이번 조사에서 **직접 확인하지 못했습니다.** 일반적인 클라우드 네트워킹(라우트 테이블, VPN/전용선 연결)으로 구성하며, 네트워크팀과 확인이 필요합니다.

# 4. Serverless compute 자세히

- 서버리스 컴퓨트는 **Databricks가 관리하는** 컴퓨트 플레인에서 실행됩니다. 계정 관리자가 **NCC(네트워크 연결 구성)** 등으로 내 리소스로의 보안 연결을 구성합니다. (→ NCC 용어 글)
- 서버리스가 내 리소스에 접속하는 방식은 리소스의 위치·구성·유형에 따라 **S3 게이트웨이 엔드포인트, 사설 IP, 공인 IP** 중 하나입니다. 같은 리전의 S3·DynamoDB를 제외한 다른 리소스에는 **공인 IP**로 접속합니다. → 방화벽에서 허용해야 하는 **아웃바운드 IP** (→ 서버리스 아웃바운드 IP 용어 글)
- 방화벽 없이 **전용·비공개 연결**이 필요하면 IP 허용 목록 대신 **outbound Private Link**(NCC의 private endpoint)를 쓰라고 안내합니다.
- 서버리스가 고객 리소스에 접속할 때 **네트워킹 비용**이 청구됩니다.
- 서버리스는 **IPv4로만** 접속하며, IPv6로만 접근 가능한 리소스에는 접속할 수 없습니다.

# 5. 이것이 "접속 Source IP"에 미치는 영향

| 질문 | Classic | Serverless |
|---|---|---|
| 외부 서버가 보는 Source IP | **내가 구성한 경로**에 따라 결정 (NAT의 공인 IP, 또는 사설 경로면 VPC 사설 IP/그 앞단 NAT IP) | Databricks가 **게시하는 아웃바운드 IP 대역**(공인) |
| 고정 IP | 내가 **NAT/LB 등으로 고정**할 수 있음 | 게시된 대역이며 **주기적으로 바뀔 수 있음** |
| 사설망 직접 연결 | 내 VPC 라우팅(peering, VPN, 전용선 등)으로 가능 | **Private Link(NCC) 등 별도 구성** 필요 |

# 6. 흔한 오해

| 오해 | 실제 |
|---|---|
| "Databricks의 Source IP는 하나로 정해져 있다" | 컴퓨트 종류·네트워크 구성에 따라 **다름** |
| "Classic 클러스터는 공인 IP가 있어서 인터넷에 직접 나간다" | 노드에 공인 IP 없음. 외부로는 **내 라우팅(NAT 등)** 을 거침 |
| "Private Link를 설정하면 SFTP도 자동으로 사설로 간다" | Private Link는 **특정 연결 구간**용. 목적지 접속 경로는 별도 구성 필요 (→ NCC 글) |
| "서버리스는 내 VPC 안에서 돈다" | **Databricks 관리 영역**에서 실행 |

# 7. 한 줄 정리

- **컴퓨트 플레인 = 내 코드가 실행되는 곳.** Classic은 **내 VPC**, Serverless는 **Databricks 관리 영역**.
- 접속 Source IP는 **Classic이면 내가 구성한 라우팅/NAT**, **Serverless면 Databricks가 게시한 아웃바운드 IP**에 따라 결정된다.
- 외부(사내망 등)와의 사설 연결은 Classic은 **내 네트워크 구성**, Serverless는 **NCC + Private Link** 등으로 따로 만든다.

# 관련 용어
NCC(Network Connectivity Configuration), 서버리스 아웃바운드 IP, SSH reverse tunnel, Databricks SFTP 커넥터

## 🔗 참고
- [Classic compute plane networking](https://docs.databricks.com/aws/en/security/network/classic/)
- [Serverless compute plane networking](https://docs.databricks.com/aws/en/security/network/serverless-network-security/)
- [VPC peering](https://docs.databricks.com/aws/en/security/network/classic/vpc-peering)
