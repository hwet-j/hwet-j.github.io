---
title:  "[DB] OracleDB 조인"  
excerpt: "JOIN"

categories: # 분류하고싶은 카테고리 입력
  - ORACLEDB
tags:     # 중요 키워드
  - [OracleDB, JOIN]

toc: true
toc_sticky: true

date: 2023-06-17
last_modified_at: 2023-06-19
---


인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1]()
{: .notice--warning}


## JOIN

### EQUI JOIN 

동일한 컬럼을 명시해줘서 두개의 테이블을 연결 시킨다.

```sql 
SELECT *
FROM emp, dept
WHERE emp.deptno = dept.deptno;
```

두 개의 테이블을 FROM절에서 선언하고 WHERE절에서 공통되는 컬럼을 묶어준다. 

이렇게 두 테이블을 연결하면 공통되는 컬럼이 있는 데이터만 연결되지만, 각 테이블 마다 1개씩 존재하며 나머지 컬럼들도 각자의 테이블에 존재한다.

그러므로 공통되는 컬럼을 SELECT절에서 사용하고 싶다면 테이블과 함께 명시해줘야한다. (공통되지 않는 컬럼. 즉,1개만 존재하는 컬럼은 명시하지 않아도 무방함)

### INNER JOIN 

FROM절에 테이블과 테이블 사이에 INNER JOIN을 명시해서 사용하며, ON 또는 USING절과 함께 사용된다. 

INNER의 경우에는 생략가능하다. JOIN의 기본값이 INNER이기 때문이다. 

> ON을 사용한 INNER JOIN

```sql 
SELECT a.deptno, ename, dname, loc
FROM emp a JOIN dept b
ON a.deptno = b.deptno
```

해당 INNER JOIN은 EQUI JOIN과 매우 흡사하다. ON을 사용하게되면 EQUI JOIN과 마찬가지로 공통 컬럼이 2개 존재하게 된다.

그러므로 공통컬럼의 경우에는 SELECT절에 사용하고 싶다면 테이블을 같이 명시해줘야한다. 

> USING을 사용한 INNER JOIN










***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🧐

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}