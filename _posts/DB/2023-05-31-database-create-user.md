---
title:  "[DB] OracleDB 계정생성, 권한부여"  
excerpt: "테이블스페이스 생성, 사용자생성...."

categories: # 분류하고싶은 카테고리 입력
  - DATABASE
tags:     # 중요 키워드
  - [OracleDB]

toc: true
toc_sticky: true

date: 2023-05-31
last_modified_at: 2023-06-03
---


인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1](https://myjamong.tistory.com/218)
{: .notice--warning}


## OracleDB

### 계정 설정

DB에 계정을 새로 생성하기 위해서는 관리자 계정에 들어가야 만들 수 있다.

#### 관리자 계정 및 교육용 계정

- 관리자 계정 (SYS, SYSTEM)

`SYS`는 ORACLE DB관리자 계정으로 super user입니다. 비밀번호는 Oracle 설치 시에 설정합니다.

Oracle 시스템의 기반이 되는 Data dictionary 소유자이며, DB 생성과 삭제도 가능하다. Oracle 시스템의 총 관리자라고 보면 되고, SYSDBA 권한을 가진다.

관리자 계정에는 `SYSTEM`도 있는데 SYS와 마찬가지로 비밀번호는 설치 시에 설정한다. 

SYS와 유사한 권한을 가지고 있지만 DB생성과 삭제는 불가능하다. 

- 교육용 계정 (HR, SCOTT)

오라클에는 처음 오라클을 사용하는 사용자를 위해서 존재하는 계정이 있다. 

`HR`계정의 경우에는 관리자 계정에서 계정의 잠금을 해제하고 비밀번호를 설정해서 사용하면 되지만,

`SCOTT`의 경우에는 처음부터 입력되어있지 않고, 오라클을 설치한 폴더에 있는 파일 scott.sql을 실행해 주면됩니다.

잠금해제와 파일실행은 관리자 계정에 접속한 상태에서 진행해야한다.

```sql
-- 관리자 계정 접속
CONN /as sysdba

-- HR계정 잠금 해제
ALTER USER hr ACCOUNT unlock;

-- HR계정 비밀번호 설정(변경)
ALTER USER hr INDENTIFIED BY hr;

-- SCOTT 파일 실행
@C:\oraclexe\app\oracle\product\11.2.0\server\rdbms\admin\scott.sql

-- 접속중인 계정 확인하기 (SCOTT파일이 실행되면 자동으로 SCOTT계정에 접속함)
SHOW USER
```

> 관리자 계정 접속 (CONN 명령어는 CONNECT로 바꿔도 상관없다)
>> - CONN /as sysdba  : SYSDBA권한으로 접속 (SYS계정 접속)
>> - CONN sys/비밀번호  : SYS계정 접속
>> - CONN system/비밀번호  : SYSTEM계정 접속 

> SQL 파일실행
> > 먼저 scott.sql파일을 찾아야 하는데 각자 설치한 경로에 따라 다르기 때문에 기본 설치 경로로 예를들면, <br>
> > C:\oraclexe\app\oracle\product\11.2.0\server\rdbms\admin\scott.sql 입니다.<br>
> > 파일 실행 방법은 "@" 뒤에 파일의 경로,확장자명까지 입력해주면 된다.

#### 사용자 계정 생성하기 

> 기본 문법

```sql
-- 계정 생성 및 비밀번호 설정
CREATE USER 계정명 IDENTIFIED BY 비밀번호;

-- 계정 비밀번호 변경
ALTER USER 계정명 INDENTIFIED BY 비밀번호;
```

> 예제

```sql
-- ora_user라는 이름의 계정 비밀번호 1234 설정
CREATE USER ora_user IDENTIFIED BY 1234;

-- ora_user 비밀번호 hong으로변경
ALTER USER ora_user INDENTIFIED BY hong;
```

#### 권한 부여

> 기본 문법

```sql 
-- 권한 부여
GRANT 시스템권한 | ROLE TO 계정명;

-- 권한 회수
REVOKE 시스템권한 | ROLE FROM 계정명;
```

오라클 시스템 권한은 시스템 권한, Role 두가지 종류로 나뉜다.

시스템 권한은 사용자(계정)가 데이터베이스에서 특정 작업을 수행할 수 있도록 한다. 

ROLE은 권한들의 집합이다. 롤을 사용하면 권한 부여를 쉽게 부여하고, 회수할 수 있다.

또한, 롤은 처음 생성되어 있기도 하지만 사용가 직접 생성할 수 있다.

권한을 부여,회수하는 명령어는 `GRANT, REVOKE`로 가능하다.

>  시스템 권한 종류 (일부)

- CREATE USER
  - 유저 생성 권한
- SELECT ANY TABLE
  - 모든 유저의 테이블 조회 권한
- CREATE ANY TABLE
  - 모든 유저의 테이블 생성 권한
- CREATE SESSION
  - 계정 접속 권한 
- CREATE TABLE
  - 테이블 생성 권한
- SYSDBA
  - DBA 권한

> Role 종류(사용자 생성 Role 제외) - SELECT * FROM role_sys_privs WHERE role = '롤이름';

- CONNECT 
  - CREATE SESSION
- RESOURCE
  - CREATE SEQUENCE, CREATE TRIGGER, CREATE CLUSTER, CREATE PROCEDURE, CREATE TYPE, CREATE OPERATOR, CREATE TABLE, CREATE INDEXTYPE
- DBA
  - 관리자에게 권한이 있는 모든 시스템 권한(데이터베이스 관리자에게만 부여)


> 예제

```sql
-- Role 권한 3개 부여
GRANT connect, resource, dba TO ora_user;

-- dba 권한 회수
REVOKE dba FROM ora_user;
```

#### 계정 생성 확인 및 삭제

> 계정 생성 확인

```
SELECT * FROM all_users;
```

> 계정 삭제

```
DROP USER 계정명 CASCADE;
```


--- 

#### 테이블스페이스 생성(필요에따라)

테이블스페이스는 하나의 데이터베이스 안에 가장 큰 논리적 저장공간으로 업무의 단위나 사용 용도에 따라
여러개의 Tablespace로 분리하여 관리되고 Segment(오브젝트)라는 논리적 저장공간의 집합이기도 하다.

데이터 저장단위는 물리적(파일), 논리적 단위 (데이터 블록->익스텐트->세그먼트->테이블스페이스)로 나눌수 있다.

여기서 가볍게 설명하고 다음에 따로 다루도록 하겠다.

> TABLESPACE 생성

CREATE TABLESPACE tablespace명
DATAFILE '경로/파일명.dbf'
SIZE 초기용량
[AUTOEXTEND ON ]  ==> 자동 증가 여부
[NEXT 추가용량]	   ==> 증가시 추가되는 용량
[MAXSIZE unlimited]; ==> 최대 사이즈 설정
==> 경로 폴더에 파일명.dbf라는 이름으로 생성되며 초기 크기는 초기용량으로 설정했으며 용량이 꽉찰것을 대비하여
자동 증가 옵션을 추가하여 5M씩 올리도록 해준다. ( 자동증가 옵션은 말 그대로 옵션이다 작성하지 않아도 무방하다)

> TABLESPACE 삭제

```sql
DROP TABLESPACE myts  
including contents AND datafiles;
```


***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}