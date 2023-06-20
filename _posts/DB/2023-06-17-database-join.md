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

JOIN의 종류에는 크게 INNER JOIN과 OUTER JOIN이 있다.

INNER JOIN은 공통된 값이 있는 두 개의 테이블을 결합하는 방식이다. INNER JOIN을 수행하면 연결된 테이블에서만 일치하는 행이 선택됩니다. 

OUTER JOIN은 INNER JOIN과 달리 한 테이블의 값이 다른 테이블에 일치하지 않더라도 해당 행을 포함시키는 방식입니다. 

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

### NON EQUI JOIN 

두 테이블 간의 동등한 조건을 열을 기준으로 하지 않고, 다른 조건을 사용하여 조인을 한다. 

일반적으로 비교연산자 (>, <, >=, <=)를 사용하여 조인의 조건을 정의하며 비교조건은 일치하지 않는 값을 가진 행도 선택가능하다. 

``` 
SELECT *
FROM emp a JOIN dept b
ON a.empno > b.deptno AND b.deptno = a.deptno;
```

위와 같이 비교연산자를 사용하여 JOIN의 조건의 설정이 가능하며, ON에는 하나의 조건이 아닌 여러개의 조건이 올 수 있음을 기억하자.


### INNER JOIN 

FROM절에 테이블과 테이블 사이에 INNER JOIN을 명시해서 사용하며, ON 또는 USING절과 함께 사용된다. 

INNER의 경우에는 생략가능하다. JOIN의 기본값이 INNER이기 때문이다. 

> ON을 사용한 INNER JOIN

```sql 
SELECT a.deptno, ename, dname, loc
FROM emp a JOIN dept b
ON a.deptno = b.deptno;
```

해당 INNER JOIN은 EQUI JOIN과 매우 흡사하다. ON을 사용하게되면 EQUI JOIN과 마찬가지로 공통 컬럼이 2개 존재하게 된다.

그러므로 공통컬럼의 경우에는 SELECT절에 사용하고 싶다면 테이블을 같이 명시해줘야한다. 

> USING을 사용한 INNER JOIN

```sql 
SELECT deptno, ename, dname, loc
FROM emp a JOIN dept b
USING(deptno);
```

ON을 사용한 INNER JOIN과 동일하지만, 공통된 컬럼의 데이터가 하나로 병합되기 때문에 테이블을 명시하여 공통컬럼을 사용할 수 없다.

또한, ON 과 달리 USING은 조건을 하나만 설정할 수 있으므로 하나의 컬럼과 하나의 컬럼을 기준으로 병합하고 싶을 때 사용한다. 

### NATURAL JOIN

```sql 
SELECT deptno, ename, dname, loc
FROM emp a NATURAL JOIN dept b;
```

NATURAL JOIN의 경우에는 사용자가 어떤 공통된 컬럼을 사용할지 설정하는 것이 아닌 공통된 컬럼명이 공통의 내용을 가진다면 알아서 인식하여 그 내용만 사용하겠다는 의미이다.

이때, 공통컬럼이 하나가 아니라 여러개여도 그 모두가 합쳐지며 USING과 마찬가지로 공통된 컬럼은 하나의 컬럼으로 합쳐져 테이블을 명시하여 사용이 불가능하다.

### OUTER JOIN

OUTER JOIN의 경우는 emp, dept 테이블은 이해하기에 그렇게 좋은 데이터는 아닌듯 하여 새로운 데이터를 사용하여 설명하겠습니다.

<p><details>
<summary style="color:red;">테스트를 위한 테이블 생성 코드</summary><!-- summary 아래 한칸 공백 필요 -->

<pre>
<code>
-- "고객" 테이블 생성
CREATE TABLE 고객 (
  고객ID NUMBER,
  이름 VARCHAR2(50),
  CONSTRAINT pk_고객 PRIMARY KEY (고객ID)
);

-- "주문" 테이블 생성
CREATE TABLE 주문 (
  주문ID NUMBER,
  고객ID NUMBER,
  주문날짜 DATE,
  CONSTRAINT pk_주문 PRIMARY KEY (주문ID),
  CONSTRAINT fk_주문_고객 FOREIGN KEY (고객ID) REFERENCES 고객 (고객ID)
);

-- "고객" 테이블에 데이터 삽입
INSERT INTO 고객 (고객ID, 이름) VALUES (1, 'John');
INSERT INTO 고객 (고객ID, 이름) VALUES (2, 'Emily');
INSERT INTO 고객 (고객ID, 이름) VALUES (3, 'Michael');
INSERT INTO 고객 (고객ID, 이름) VALUES (4, 'Sarah');

-- "주문" 테이블에 데이터 삽입
INSERT INTO 주문 (주문ID, 고객ID, 주문날짜) VALUES (1, 1, TO_DATE('2022-01-01', 'YYYY-MM-DD'));
INSERT INTO 주문 (주문ID, 고객ID, 주문날짜) VALUES (2, 2, TO_DATE('2022-02-15', 'YYYY-MM-DD'));
INSERT INTO 주문 (주문ID, 고객ID, 주문날짜) VALUES (3, 1, TO_DATE('2022-03-10', 'YYYY-MM-DD'));

</code>
</pre>

</details></p>

#### LEFT OUTER JOIN

왼쪽 테이블의 모든 행과 오른쪽 테이블의 일치하는 행을 반환합니다. 

오른쪽 테이블에 일치하는 행이 없을 경우에도 왼쪽 테이블의 모든행이 결과에 포함되며, 오른쪽에 없는 값은 NULL로 표현됩니다.

```sql 

```







***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🧐

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}