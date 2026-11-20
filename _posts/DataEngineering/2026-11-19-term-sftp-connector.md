---
title:  "[DE/용어] Databricks SFTP 커넥터"
excerpt: "SFTP 서버의 파일을 Unity Catalog 연결과 Auto Loader로 증분 수집하는 방법과 네트워크 사전 확인 사항"

categories:
  - DataEngineering
tags:
  - [Glossary, Databricks, SFTP, Lakeflow Connect, Auto Loader]

toc: true
toc_sticky: true
published: true

date: 2026-11-19
last_modified_at: 2026-11-19
---

용어 정리 글입니다. 테이블명·컬럼명은 모두 **예시**입니다.    
공식 문서를 참고하여 정리하였습니다. (확인일: 2026-11-19)    
[Ingest files from SFTP servers](https://docs.databricks.com/aws/en/ingestion/sftp)
{: .notice--warning}

# 0. 이 글을 읽기 전에

SFTP 서버의 파일을 Databricks로 가져오는 공식 방법을 다룹니다. **컴퓨트 플레인(Classic vs Serverless)** 글의 "접속 Source IP가 컴퓨트에 따라 다르다"를 알고 계시면 "네트워크 사전 확인" 부분이 쉽게 이해됩니다. (AWS 문서 기준)

# 1. 한 줄 정의

**Lakeflow Connect의 SFTP 커넥터**는 Auto Loader 기능을 확장해서, SFTP 서버에서 **안전하게 증분으로** 파일을 수집하고 **Unity Catalog로 거버넌스**하는 기능입니다.

# 2. 주요 기능 (공식 문서)

- **Private key**와 **비밀번호** 기반 인증
- **증분 파일 수집**(이미 처리한 파일은 다시 처리하지 않음)과 **exactly-once 보장**
- 자동 스키마 추론·진화, 데이터 rescue
- Unity Catalog 기반 **수집 및 자격 증명 거버넌스**
- 지원 형식: JSON, CSV, XML, PARQUET, AVRO, TEXT, BINARYFILE, ORC
- **패턴·와일드카드**로 일부 파일만 선택
- **모든 컴퓨트 유형**(Lakeflow 파이프라인, Databricks SQL, 서버리스, 클래식)에서 사용 가능 — **Databricks Runtime 17.3 이상**

# 3. 사전 요건

- Unity Catalog가 활성화된 워크스페이스
- SFTP **connection**을 만들 `CREATE CONNECTION` 권한. 기존 연결을 쓸 때는 클러스터 접근 모드에 따라 `MANAGE CONNECTION`(Dedicated) 또는 `USE CONNECTION`(Standard)
- 컴퓨트: Databricks Runtime **17.3 이상** (`databricks.connection` 옵션으로 연결을 명시하려면 **18.2 이상**)

# 4. ⚠️ 네트워크 사전 확인 (접속 문제는 대부분 여기서부터 확인합니다)

공식 문서는 **먼저 SFTP 서버가 Databricks 컴퓨트 환경에서 접근 가능한지** 확인하라고 안내합니다.

1. **원격 서버가 워크스페이스에 설정된 VPC에서 접근 가능**한지 확인
2. **SSH 규칙이 허용**하는지 확인: 
   - **Classic 컴퓨트**: **Databricks VPC의 IP 대역**
   - **Serverless 컴퓨트**: **아웃바운드 IP** (→ 서버리스 아웃바운드 IP 글)
3. Classic에서 **고정 IP**가 필요하면, **로드 밸런서, NAT 게이트웨이, 인터넷 게이트웨이 등**을 Databricks 컴퓨트가 배치된 서브넷에 연결해서 **허용 목록에 올릴 수 있는 안정적인 공인 IP**를 공유하게 합니다. (VPC peering 문서 참고)

즉 **SFTP 서버 입장에서 보이는 접속 IP는 컴퓨트 종류와 네트워크 구성에 따라 달라지므로**, 서버 방화벽 담당자에게 어떤 IP를 허용해 달라고 해야 하는지 **먼저 정해야** 합니다.

# 5. 연결(Connection) 만들기

SFTP 자격 증명은 **Unity Catalog connection**에 저장합니다. 인증 방식은 두 가지입니다.

| 방식 | 비고 |
|---|---|
| **PEM private key** | **권장**. 서버에서는 **최소 권한**(예: 읽기 전용 일반 사용자, root 아님) 계정 사용을 권장 |
| 사용자명 + 비밀번호 | 가능하지만 private key를 권장 |

## 호스트 키 지문(fingerprint) 검증

- **Enforce host key fingerprint**를 켜면, 서버의 공개 키가 기대하는 **SHA-256 지문**과 일치할 때만 연결합니다. (중간자 공격 방지)
- 지문은 서버 관리자에게 받거나 CLI로 얻을 수 있고, **Test and create connection → Test**의 **오류 메시지**에서도 확인할 수 있습니다. (예: `ECDSA key fingerprint is SHA256:...`)
- 검증을 끄면 **지문이 달라도 연결이 진행**되므로 **끄기 전에 네트워크 관리자와 확인**하라고 문서가 경고합니다.

## SQL 예시 (PEM 키)

```sql
CREATE CONNECTION my_sftp_connection
TYPE sftp
OPTIONS (
  host 'sftp.example.internal',
  user secret('my_secret_scope', 'my_sftp_username'),
  pem_private_key secret('my_secret_scope', 'my_sftp_private_key'),
  port '22',
  pem_key_passphrase secret('my_secret_scope', 'my_sftp_private_key_passphrase'),   -- 선택
  key_fingerprint 'SHA256:ExampleFingerprint...'
);
```

- 자격 증명은 **코드에 직접 쓰지 말고 secret scope**에서 참조하는 것을 권장합니다.

# 6. 파일 읽기 (Auto Loader)

```python
df = (spark.readStream.format("cloudFiles")
  .option("cloudFiles.schemaLocation", "<스키마 저장 경로>")        # 클라우드 스토리지 경로
  .option("cloudFiles.format", "csv")                               # 지원 형식 중 하나
  .load("sftp://<username>@<host>:<port>/<absolute_path_to_files>") # 루트(/)부터의 절대 경로
  .writeStream
  .format("delta")
  .option("checkpointLocation", "<체크포인트 경로>")
  .trigger(availableNow=True)
  .table("<대상 테이블>"))
df.awaitTermination()
```

## 연결을 지정하는 두 가지 방법

| 방법 | 설명 |
|---|---|
| **자동(automatic)** | URI의 `<host>`·`<username>`과 일치하는 connection을 자동 선택. 여러 개면 **처음으로 연결에 성공한 것** 사용 |
| **명시(explicit, 권장, DBR 18.2+)** | `.option("databricks.connection", "<연결 이름>")`. 이때 URI의 host·port는 저장된 연결과 **일치**해야 하고 username은 생략 가능 |

- 경로는 **SFTP 서버의 루트(`/`)부터의 절대 경로**여야 합니다.
- `availableNow` 트리거는 **현재 있는 파일을 모두 처리하고 종료**하는 배치형 동작입니다.
- 스키마·체크포인트 저장 경로는 **클라우드 스토리지 경로**입니다.

# 7. 자주 하는 오해와 문제

| 증상/오해 | 점검 |
|---|---|
| "연결 생성 시 timeout" | **네트워크 경로/방화벽**(SSH 22 허용, 컴퓨트 종류별 Source IP) 확인 |
| "서버리스에서 안 되는데 Classic은 된다" | 서버리스는 **공인 아웃바운드 IP**, Classic은 **내 VPC 경로**로 나가서 서버가 보는 IP가 다름 |
| "지문 불일치 오류" | 서버 키가 바뀌었는지/지문을 정확히 입력했는지. 함부로 검증을 끄지 말 것 |
| "연결이 여러 개 매칭된다" | 자동 해석은 첫 성공 연결을 사용 → 명시 방식(DBR 18.2+)으로 고정 |
| "비밀번호를 코드에 적어도 되나" | **secret scope** 사용 권장 |

# 8. 한 줄 정리

- SFTP 커넥터 = **Auto Loader 기반 증분 수집 + Unity Catalog connection**, DBR **17.3+**, 모든 컴퓨트에서 사용.
- 인증은 **PEM 키 권장**, **호스트 키 지문 검증**을 켠 채로 사용, 서버 계정은 **최소 권한**.
- 가장 먼저 확인할 것은 **네트워크**: **컴퓨트 종류(Classic/Serverless)에 따라 서버가 보는 Source IP가 다르다.**
- 인바운드가 막힌 서버라면 **사설 경로 구성**(Classic + 사설 라우팅, NCC Private Link, SSH reverse tunnel 등)이 필요하다.

# 관련 용어
컴퓨트 플레인(Classic vs Serverless), 서버리스 아웃바운드 IP, NCC, SSH reverse tunnel

## 🔗 참고
- [Ingest files from SFTP servers](https://docs.databricks.com/aws/en/ingestion/sftp)
