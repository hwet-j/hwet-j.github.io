---
title:  "[DE/용어] SSH reverse tunnel (역방향 SSH 터널)"
excerpt: "방화벽의 인바운드를 열지 않고 사내 리소스를 Databricks에서 쓰게 하는 연결 방식"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, Network, SSH, On-premises]

toc: true
toc_sticky: true
published: true

date: 2026-11-19
last_modified_at: 2026-11-19
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-19)    
[Connect to on-premises resources using an SSH reverse tunnel](https://docs.databricks.com/aws/en/security/network/serverless-network-security/ssh-reverse-tunnel)
{: .notice--warning}

# 0. 이 글을 읽기 전에

"사내 서버의 **인바운드는 막혀 있는데** Databricks가 그 서버에 접속해야 한다"는 상황에서 공식 문서가 소개하는 방식입니다. **컴퓨트 플레인**, **NCC** 글을 먼저 보시면 좋습니다. (AWS 문서 기준)

> 이 방식은 문서에서 **self-managed(자체 구성·운영) 솔루션**이라고 명시합니다. 프록시 VM과 온프레미스 터널 호스트를 직접 구성·유지해야 합니다.
{: .notice--warning}

# 1. 한 줄 정의

**온프레미스(사내) 쪽 호스트가 클라우드의 프록시 VM으로 "바깥쪽 방향" SSH 연결을 먼저 열어 두고**, Databricks가 그 프록시를 통해 온프레미스 리소스에 도달하게 하는 방식입니다. 방화벽에서 **인바운드를 열 필요가 없습니다.**

# 2. 왜 "리버스(역방향)"인가

```
[일반 방식]  Databricks ───────────────────────► 사내 서버     ❌ 사내 방화벽이 인바운드를 차단

[역방향]     사내 터널 호스트 ──(먼저 아웃바운드 SSH 연결)──► 클라우드 프록시 VM
              Databricks ──► 프록시 VM ──(이미 열린 터널을 따라)──► 사내 리소스  ✅
```

- 사내 방화벽 입장에서는 "내부 → 외부로 나가는 SSH(22)"만 필요합니다.
- **돌아오는 트래픽은 이미 확립된 경로**를 따라 사내 리소스로 갑니다.

# 3. 구성 요소

## 최소 구성 (공식 문서 기준)

| 구성 요소 | 역할 |
|---|---|
| **온프레미스 터널 호스트** | `autossh`로 프록시 VM에 **아웃바운드 SSH 연결**을 유지 |
| **클라우드 프록시 VM** (1대 이상) | 터널을 받고(`sshd`), `socat`으로 리소스 포트를 네트워크 인터페이스에 노출 |
| **Databricks → 프록시 VM 경로** | Classic: **VPC peering**, Serverless: **private endpoint(NCC private endpoint rule)** — 두 종류를 모두 쓰면 둘 다 구성 |
| **온프레미스 ↔ 클라우드 전용 회선** | 터널 호스트가 프록시 VM의 사설 IP로 SSH할 수 있게 하는 **전용선(Direct Connect) 또는 VPN** |

- 개발·테스트 단계에서는 **프록시 VM 1대, 로드 밸런서 없이**도 충분합니다.

## 운영용(고가용성) 추가 구성

- **프록시 VM 여러 대**(최소 2대 권장)
- 프록시 VM 앞단의 **로드 밸런서** (안정적인 엔드포인트, 터널 장애 시 자동 failover)
- 각 VM의 **HTTP 헬스 체크 서비스**: 터널이 살아있으면 **200**, 아니면 **503**을 반환. 단순 TCP 확인은 `socat`만 확인하므로 **터널이 죽은 것은 못 잡음**
- 서버리스 사용 시 로드 밸런서 앞에 **프라이빗 연결 서비스**(AWS: VPC Endpoint Service)

# 4. 프록시 VM 위의 세 가지 서비스

| 서비스 | 역할 |
|---|---|
| `sshd` | 온프레미스에서 오는 리버스 터널을 받고, **localhost에 터널 리스너**를 둠 (예: `localhost:13306`) |
| `socat` | VM의 **네트워크 인터페이스 포트 ↔ 터널 리스너**를 연결 (예: `NIC:3306 → localhost:13306`) |
| HTTP 헬스 체크 | 터널 활성 시 200, 비활성 시 503 |

온프레미스 터널 호스트는 프록시 VM마다 `autossh` 프로세스를 하나씩 실행하고, **하나의 SSH 연결에 여러 포트 포워딩(`-R`)** 을 담을 수 있습니다. 운영에서는 **systemd 서비스(`Restart=always`)** 로 돌려야 하며, 사용자가 로그아웃하면 끊기는 **인터랙티브 터널은 운영에 부적합**합니다.

# 5. Databricks에서 사용하는 방법

연결 설정에서 **온프레미스 리소스 주소 대신 로드 밸런서(또는 프록시)의 프런트엔드 IP**를 지정합니다. 문서의 예시는 MySQL 연결입니다(다른 DB는 연결 유형·포트·드라이버만 바꿈).

```sql
CREATE CONNECTION mysql_onprem TYPE mysql
OPTIONS (
  host '<lb-frontend-ip>',
  port '3306',
  user '<db-user>',
  password '<db-password>'
);
```

- **Classic 컴퓨트**: 프록시 hub VPC와 Databricks 워크스페이스 VPC 간 **peering이 활성**인지 확인 후, 로드 밸런서 프런트엔드 **사설 IP**로 연결.
- **Serverless 컴퓨트**: 계정 콘솔에서 **NCC를 만들고 private endpoint rule**을 추가(서비스 리소스 ID 입력).

# 6. 이 방식이 맞는 경우

| 상황 | 적합성 |
|---|---|
| 사내 서버가 **인바운드 차단**이고 **아웃바운드 SSH(22)** 는 허용 가능 | ✅ |
| 클라우드와 사내망 사이에 **전용선/VPN**이 있음 | ✅ (필수 전제) |
| 사내 서버가 **공인망으로 접근 가능**하고 IP 허용이 쉬움 | 굳이 필요 없음 (서버리스 아웃바운드 IP 허용 방식) |
| 직접 **VM·터널 운영 부담**을 질 수 없음 | ⚠️ 부담이 큼 (self-managed) |

# 7. 한계와 고려사항

- **직접 운영해야 함**: 프록시 VM, 터널 호스트, 로드 밸런서, 헬스 체크의 설치·모니터링·패치.
- **터널이 끊기면 연결이 끊김** → 헬스 체크와 자동 재연결(`autossh`, systemd) 필수.
- 보안 검토: 사내 정책이 **아웃바운드 SSH**를 허용하는지, **포트 포워딩**이 허용되는지 확인.
- 전용선/VPN이 **전제 조건**이므로, 없으면 이 방식도 쓸 수 없음.

# 8. 한 줄 정리

- SSH reverse tunnel = **사내 호스트가 먼저 바깥으로 SSH를 열고, 그 터널을 통해 Databricks가 사내 리소스에 접근**하는 방식. **인바운드를 열 필요가 없다.**
- 필수: **터널 호스트(autossh) + 프록시 VM(sshd, socat) + Databricks→프록시 경로 + 전용선/VPN**. 운영에는 **LB + 헬스 체크 + 다중 VM** 권장.
- **자체 구성·운영** 솔루션이다.

# 관련 용어
NCC, 컴퓨트 플레인(Classic vs Serverless), 서버리스 아웃바운드 IP

## 🔗 참고
- [Connect to on-premises resources using an SSH reverse tunnel](https://docs.databricks.com/aws/en/security/network/serverless-network-security/ssh-reverse-tunnel)
