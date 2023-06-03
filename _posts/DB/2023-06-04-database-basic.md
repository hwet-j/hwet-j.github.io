---
title:  "[DB] OracleDB 기본문법"  
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


## OracleDB 기본문법

### SELECT

> SELECT문의 문법

```sql 
SELECT [DISTINCT] 컬럼명 [AS ALIAS명]
FROM 테이블명
[WHERE 조건]
[GROUP BY 그룹으로묶을컬럼명]
[HAVING 묶은 그룹 조건]
[ORDER BAY 컬럼명 [ACS | DESC]; 
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





***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}