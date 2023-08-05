---
title:  "[DB/OracleDB] Sequence"  
excerpt: "시퀀스란 무엇이며 언제쓰이나"

categories: # 분류하고싶은 카테고리 입력
  - DataBase
tags:     # 중요 키워드
  - [OracleDB]

toc: true
toc_sticky: true

date: 2023-06-07
last_modified_at: 2023-06-14
---


인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1](https://mine-it-record.tistory.com/61)
{: .notice--warning}


## 시퀀스 : SEQUENCE

오라클 DB에서 시퀀스(Sequence)는 일련번호를 생성하는 객체입니다. 

시퀀스는 고유한 번호를 설정한 조건에 따라 자동으로 생성하여 데이터베이트의 테이블의 열에 할당하는데 사용된다.

`일반적으로 고유식별자(PK:Primary Key)값을 설정하는데 사용`되며, 여러 테이블에서 사용가능합니다. 

시퀀스는 생성시에 시작 값, 증가/감소 값, 최대 값 등을 설정가능하며, 이후에 수정도 가능하다.

하지만 시작 값(현재 시퀀스값)은 수정 불가능하기 때문에 사용 시 주의하여 사용해야 합니다. 



### 🔹 Sequence의 생성 

> Sequence 문법

```sql 
CREATE SEQUENCE sequence_name
[START WITH start_value]
[INCREMENT BY increment_value]
[MINVALUE min_value]
[MAXVALUE max_value]
[CYCLE | NOCYCLE]
[CACHE cache_value | NOCACHE];
```

> Sequence 키워드

```
sequence_name : 시퀀스의 이름입니다.
start_value : 시퀀스의 시작 값입니다. (DEFAULT : 1)
increment_value : 시퀀스의 증가 값입니다. (DEFAULT : 1)
min_value : 시퀀스의 최소 값입니다. (DEFAULT : 1)
max_value : 시퀀스의 최대 값입니다.
CYCLE : 시퀀스가 최대 값에 도달하면 다시 최소 값부터 시작하는 것을 의미합니다.
NOCYCLE :시퀀스가 최대 값에 도달하면 오류를 발생시킵니다. (DEFAULT)
cache_value : 시퀀스 값을 캐시하는 개수입니다. (DEFAULT : 20 ==> CACHE 20)
```

> Sequence 생성 예시

```sql 
CREATE SEQUENCE name_seq
START WITH 1
INCREMENT BY 1
MINVALUE 1
MAXVALUE 9999
NOCYCLE
CACHE 20;
```

무조건은 아니지만 일반적으로 시퀀스명은 앞이나 뒤에 seq를 붙여 시퀀스임을 명시해준다. 

### 🔹 Sequence 삭제

```sql 
DROP SEQUENCE sequence_name;
```

### 🔹 Sequence 수정

```sql 
ALTER SEQUENCE sequence_name
[INCREMENT BY n]
[MAXVALUE n | NOMAXVALUE]
[MINVALUE n | NOMINVALUE]
[CYCLE | NOCYCLE]
[CACHE n | NOCACHE];
```

> 시작번호를 수정하고 싶다면 해당 시퀀스를 삭제했다가 만드는 방법을 사용해야 한다.


### 🔹 Sequence의 사용

> Sequence 사용 예시

#### 생성된 테이블에서 INSERT문에 사용하기 

```sql
CREATE TABLE table_name (
  id NUMBER,
  name VARCHAR2(100),
  age NUMBER
);
```

id, name ,age의 컬럼을 가진 테이블을 생성

> INSERT 문에서 시퀀스 사용

```sql 
INSERT INTO employees (id, name, age) VALUES (name_seq.NEXTVAL, '이름', 30);
```

위와 같이 입력해주면 id값은 설정해주지 않았음에도 자동으로 설정된다.

여기서, id 값과 같이 입력시마다 일정한 규칙으로 증가하는 컬럼이 있는데 INSERT문을 작성할 때마다 굳이 작성해줘야 할까?

`INSERT INTO employees (name, age) VALUES ('이름', 30)` 와 같이 INSERT문에서 명시하지 않아도 자동으로 입력되도록 할 수 있는 방법을 찾아보았다.

> 테이블생성시 DEFAULT값으로 설정이 가능할까? 

```sql
CREATE TABLE table_name (
  id NUMBER DEFAULT name_seq.NEXTVAL,
  name VARCHAR2(100),
  age NUMBER
);
```

과 같이 설정해 보았다. 하지만 `ORA-00984: column not allowed here` 에러가 발생하였다.

오라클 데이터베이스에서는 시퀀스의 NEXTVAL 값을 테이블 생성 시 DEFAULT 값으로 할당이 불가능하다.

이 문제를 해결할 대안은 PL/SQL을 활용하는 방법이 있다. 

PL/SQL의 트리거(Trigger)를 사용해서 시퀀스의 NEXTVAL 값을 INSERT문 실행시 자동으로 할당할 수 있다.

```sql 
CREATE OR REPLACE TRIGGER trigger_name
BEFORE INSERT ON table_name
FOR EACH ROW
BEGIN
    :NEW.id := name_seq.NEXTVAL;
END;
/
```

위의 코드는 BEFORE INSERT 트리거를 정의하여 INSERT문이 실행되기 전에 트리거가 작동되도록 설정하는 것입니다. 

트리거 내에서 NEW.id에 name_seq.NEXTVAL 값을 할당하여 자동으로 시퀀스 값이 할당되는 것입니다.

이 트리거로 인해서 INSERT문을 사용하여 데이터를 삽입할 때, id 컬럼에 값을 명시하지 않아도 트리거가 작동하여 값이 할당됩니다.

```sql 
INSERT INTO employees (name, age) VALUES ('이름2', 40)
``` 
 
의 코드로 id값을 입력하지 않아도 정상적으로 입력됩니다.





***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}