---
title:  "[DB] OracleDB 첫걸음"  
excerpt: "테이블스페이스 생성, 사용자생성...."

categories: # 분류하고싶은 카테고리 입력
  - DATABASE
tags:     # 중요 키워드
  - [OracleDB]

toc: true
toc_sticky: true

date: 2023-05-31
last_modified_at: 2023-05-31
---


인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1](https://myjamong.tistory.com/218)
{: .notice--warning}


## OracleDB(를 기준으로 작성)

### 관리자 계정

관리자 계정에

#### SYS

ORACLE DB관리자 계정으로 super user입니다. 비밀번호는 Oracle 설치 시에 설정합니다.

Oracle 시스템의 기반이 되는 Data dictionary 소유자이며, DB 생성과 삭제도 가능하다. Oracle 시스템의 총 관리자라고 보면 되고, SYSDBA 권한을 가진다.

#### SYSTEM

SYS와 마찬가지로 비밀번호는 설치 시에 설정한다. 

SYS와 유사한 권한을 가지고 있지만 DB생성과 삭제는 불가능하다. 

운영을 위한 권한을 가진다.


### 사용자 계정 생성하기 

초기에 사용자가 만들어준 계정이 존재하지 않기 때문에 관리자 계정에 접속하여 생성해준다.

#### 1. 관리자 계정에 로그인 하는 방법

`CONNECT sys`를 입력해주면 비밀번호 입력창이 나오는데 설치 시에 만들어준 비밀번호를 입력한다. (사용자 생성에는 SYSTEM 계정에 접속해도 상관없다.)

`CONNECT sys/비밀번호`로 입력해도 접속가능하지만 보안상 좋지 않은 방식이므로 지양하는 것이 좋다.

`CONNECT`는 `CONN`으로 대체해도 된다. 

> 계정에 접속이 잘 되었는지 확인하는 방법

`SHOW USER` 을 입력하면 현재 접속되어있는 계정명을 불러온다.

#### 2. 테이블스페이스 생성(필요에따라)

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

#### 3. 계정 생성

```sql


```











***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}