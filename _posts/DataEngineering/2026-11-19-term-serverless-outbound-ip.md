---
title:  "[DE/용어] 서버리스 아웃바운드 IP (방화벽 허용 목록)"
excerpt: "서버리스 컴퓨트가 외부 서버에 접속할 때 쓰는 공인 IP를 알아내고 방화벽에 등록하는 방법"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, Network, Serverless Compute, Firewall]

toc: true
toc_sticky: true
published: true

date: 2026-11-19
last_modified_at: 2026-11-19
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-19)    
[Serverless compute firewall configuration](https://docs.databricks.com/aws/en/security/network/serverless-network-security/serverless-firewall)    
[Serverless compute plane networking](https://docs.databricks.com/aws/en/security/network/serverless-network-security/)
{: .notice--warning}

# 0. 이 글을 읽기 전에

서버리스 컴퓨트로 **방화벽이 있는 외부 서버(SFTP, DB 등)** 에 접속하려면, 서버 쪽 방화벽이 **Databricks의 접속 IP를 허용**해야 합니다. 이 글은 그 IP를 **어디서 얻고, 어떻게 관리하는지**를 다룹니다. (AWS 문서 기준, 해당 기능은 문서에서 **Public Preview**로 표기)

# 1. 한 줄 정의

서버리스 컴퓨트가 같은 리전의 S3·DynamoDB 이외의 리소스에 접속할 때 쓰는 **공인 IP 대역(CIDR)** 으로, Databricks가 **`ip-ranges.json` 파일로 게시**합니다.

# 2. 배경: 왜 IP를 허용해야 하나

- 서버리스는 같은 리전의 **S3 버킷**에는 **VPC 엔드포인트(게이트웨이 엔드포인트)** 로 접속합니다(OrgPath 조건 키로 제한 가능).
- 그 밖의 리소스(예: 외부 파일 서버, 다른 DB)에는 **공인 IP**로 접속합니다. 따라서 대상의 방화벽에서 **Databricks가 게시한 아웃바운드 IP를 허용**해야 합니다.
- 방화벽을 만들면 **Classic 컴퓨트의 접속에도 영향**을 줍니다. Classic에서 접속하는 경로의 네트워크도 **별도로 허용**해야 합니다.

# 3. IP를 찾는 방법 (공식 문서)

1. `ip-ranges.json`을 내려받는다.
2. 다음 조건에 맞는 항목만 남긴다.
   - `service` = `Databricks`
   - `type` = `outbound`
   - `region` = **워크스페이스의 리전**
   - `platform` = 사용하는 클라우드(AWS 문서에서는 `aws`)
3. 일치하는 항목의 **`ipv4Prefixes`(CIDR 블록)** 를 대상 리소스의 방화벽 허용 목록에 추가한다.

> 서버리스는 **IPv4**로 접속하며, IPv6로만 접근 가능한 리소스에는 연결할 수 없습니다.
{: .notice--info}

# 4. 가장 중요한 주의: IP는 바뀐다

공식 문서가 강조하는 점입니다.

- Databricks는 이 IP를 **시간이 지나면서 변경**합니다. 한 번 복사해서 쓰면 **언젠가 서버리스 연결이 끊깁니다.**
- 갱신은 **최대 30일에 한 번** 게시될 수 있고, **새 IP는 게시 후 60일쯤 뒤부터 활성화**될 수 있습니다. 새 리전도 주기적으로 추가됩니다.
- 따라서 **허용 목록 갱신을 자동화**하라고 안내합니다.

```
① ip-ranges.json 을 주기적으로(예: 30일마다) 가져온다
② 파일의 timestampSeconds 값을 저장해 둔 값과 비교해 변경 감지
③ 변경됐다면 내 플랫폼·리전 해당 IP가 바뀌었는지 확인
④ 방화벽 허용 목록을 갱신
⑤ 이번 파일을 저장해 다음 비교에 사용
```

# 5. 예전 "stable IP"와의 관계

- 예전에 Public Preview 고객에게 NCC로 제공하던 **legacy stable IP 목록은 폐기**(문서상 2026-05-25에 deprecated)되었고, NCC UI/API에서도 더 이상 제공되지 않습니다.
- 이를 쓰고 있었다면 **새 방식으로 이전**해야 하며, 이전 전에는 서버리스가 접근할 때 **403 Forbidden**이 날 수 있습니다.

# 6. 적용 시 고려사항

- 방화벽 규칙이 **전파되는 시간**이 필요하므로, 설정 직후 바로 테스트하면 실패할 수 있습니다.
- **전용·비공개 연결**이 필요하다면 IP 허용 대신 **outbound Private Link**를 쓰라고 안내합니다. (→ NCC 글)
- 허용해야 할 IP가 **바뀔 수 있는 대역**이므로, 인바운드를 엄격히 통제하는 환경(예: 인바운드 완전 차단 예정인 외부 서버)에서는 **공인 IP 허용 방식 자체가 맞지 않을 수 있습니다.**

# 7. 이 방식이 맞는 경우와 아닌 경우

| 상황 | 아웃바운드 IP 허용 방식 |
|---|---|
| 대상 서버가 **공인 인터넷에서 접근 가능**하고 방화벽으로 IP 제한 | 적합 |
| 대상이 **사내망(인바운드 차단)** 이고 사설 경로로만 접근 | **부적합** → NCC Private Link, SSH reverse tunnel, 또는 Classic 컴퓨트 + 사설 경로 검토 |
| 허용 IP를 정적으로 한 번만 등록하고 싶음 | **부적합** (IP가 바뀜) → 자동 갱신 필요 |

# 8. 한 줄 정리

- 서버리스가 외부 서버에 접속할 때 쓰는 **공인 IP 대역은 Databricks가 `ip-ranges.json`으로 게시**한다. (`service=Databricks`, `type=outbound`, 내 리전·플랫폼으로 필터 → `ipv4Prefixes`)
- **IP는 바뀌므로 허용 목록을 자동으로 갱신**해야 한다. (게시 ≤30일 주기, 활성화 ≥60일 뒤)
- 옛 **stable IP는 폐기**되었다.
- **사설망 전용 연결**이 필요하면 IP 허용이 아니라 **Private Link**를 쓴다.

# 관련 용어
컴퓨트 플레인(Classic vs Serverless), NCC, SSH reverse tunnel

## 🔗 참고
- [Serverless compute firewall configuration](https://docs.databricks.com/aws/en/security/network/serverless-network-security/serverless-firewall)
- [Serverless compute plane networking](https://docs.databricks.com/aws/en/security/network/serverless-network-security/)
