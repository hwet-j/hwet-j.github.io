---
title:  "[DB/OracleDB] 기본문법(DML)"
excerpt: "CRUD"

categories: # 분류하고싶은 카테고리 입력
  - DataBase
tags:     # 중요 키워드
  - [OracleDB]

toc: true
toc_sticky: true

date: 2023-06-04
last_modified_at: 2023-06-09
---


인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1](https://coding-factory.tistory.com/)
[참고2](https://m.blog.naver.com/regenesis90/222213840145)
[참고3](https://velog.io/@yunyoseob/Oracle-DB-SQL-%EA%B8%B0%EC%B4%88%EB%AC%B8%EB%B2%95-1%ED%8E%B8)
{: .notice--warning}


## OracleDB 기본문법 (DML)

`DML`은 Data Manipulation Language로 데이터 조작어입니다.

데이터베이스에서 데이터를 삽입, 조회, 업데이트 및 삭제를 하는데 사용되며 일반적으로 가장 많이 사용된다.

### SELECT (조회)

> SELECT문의 문법

```sql 
SELECT [DISTINCT] 컬럼명 [AS ALIAS명]
FROM 테이블명
[WHERE 조건]
[GROUP BY 그룹으로묶을컬럼명]
[HAVING 묶은 그룹 조건]
[ORDER BY 컬럼명 [ASC | DESC]; 
```

> SELECT문 키워드

```
SELECT : 데이터를 출력(조회)
FROM : 출력하고 싶은 테이블 이름 정의
WHERE : 데이터 출력의 조건
GROUP BY : 특정 기준으로 그룹화해주는 조건
HAVING : 그룹 함수를 포함한 조건
ORDER BY : 데이터 정렬
```

> SELECT문의 실행순서

```
FROM - WHERE - GROUP BY - HAVING - SELECT - ORDER BY 순으로 실행된다.
```

실행 순서에 맞게 진행되기 때문에 별칭을 사용할 때에도 적용시기에 맞게 작성해줘야한다.

SELECT절이 WHERE절보다 더 늦은 순서로 `SELECT column1 AS C1 FROM table1 WHERE C1 = 10`과 같은 명령어는 에러가 발생한다.

`SELECT column1 AS C1 FROM table1 WHERE column1 = 10`와 같이 사용해야 하며, WHERE절에는 SELECT에서 적용된 상태를 가져다 사용불가능하다.

<strong style="color:red">! ORDER BY 정렬기준에 null이 존재할 경우</strong>

ORACLE DB에서는 null값을 가장 큰 값으로 인식하여 정렬시에 가장 위(내림차순 시에)에 존재한다.


### INSERT (입력)

> INSERT 문법

```sql
INSERT INTO 테이블명[(컬럼명, 컬럼명...)]
VALUES(값, 값....);
```

테이블명 뒤에서 괄호안에 컬럼명을 명시해줬다면 명시해준 컬럼의 개수와 값의 개수가 동일해야 한다.

테이블명 뒤에 어떠한 컬러몀도 명시하지 않았다면 모든 컬럼에 대한 데이터를 삽입하겠다는 의미이다.

### UPDATE (수정)

> UPDATE 문법

```sql 
UPDATE 테이블명
SET 컬럼1 = 값,
컬럼2 = 값,
...
[WHERE 조건];
```

UPDATE문에서는 매우 높은 확률로 WHERE절이 작성된다. 조건없이 작성하면 설정한 테이블의 모든 데이터가 변경되기 때문이다.

### DELETE (삭제)

> DELETE 문법

```sql 
DELETE [FROM] 테이블명
[WHERE 조건];
```

DELETE문도 UPDATE문과 비슷하게 매우 높은 확률로 WHERE절이 작성되며, 조건없이 작성하면 테이블 자체를 삭제한다.


### 트랜젝션 제어 명령어 (COMMIT, ROLLBACK, SAVEPOINT)

> COMMIT

작업을 보류중인 데이터 변경사항을 영구적으로 적용 및 현재 트랜젝션을 종료함

> ROLLBACK

특정단계의 직후 부터 작업을 보류중인 데이터 변경사항을 제거하며 현재 트랜젝션을 종료하며 특정단계 직후로 되돌아간다.

특정단계는 COMMIT한 직후 또는 SAVEPOINT의 직후를 의미한다.

> SAVEPOINT

ROLLBACK할 포인트를 지정한다.

``` 
입력, 수정, 삭제를 진행하다보면 잘못 작업하는 경우가 발생하는데 이럴때 사용한다. 

만약 DELETE문을 작성하는데 WHERE절을 까먹어서 작성하지 않아 테이블 전체를 삭제했을 때, ROLLBACK 명령어로 단계를 되돌릴 수 있다.

COMMIT을 하기 이전에 현재 진행중인 사용자 내부에서 SELETE문으로 작업진행 결과를 확인 가능하지만 다른 사용자에서 확인하면 마지막에 COMMIT한 직후의 내용만 나타난다.

ROLLBACK 명령은 작은 작업 단위로 여러개의 진행 포인트를 사용자가 이름을 명시하여 특정 단계마다 설정가능하며 설정한 이름으로 명시하여 ROLLBACK 가능하다.

COMMIT을 진행하게되면 현재상태 단계에서 작업을 영구적으로 적용하며 모든 SAVEPOINT는 삭제된다.
```




***
<br>

    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}