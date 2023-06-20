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
last_modified_at: 2023-06-20
---


인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1](https://www.codeproject.com/Articles/33052/Visual-Representation-of-SQL-Joins)
{: .notice--warning}


## JOIN

JOIN의 종류에는 크게 INNER JOIN과 OUTER JOIN이 있다.

INNER JOIN은 공통된 값이 있는 두 개의 테이블을 결합하는 방식이다. INNER JOIN을 수행하면 연결된 테이블에서만 일치하는 행이 선택됩니다. 

OUTER JOIN은 INNER JOIN과 달리 한 테이블의 값이 다른 테이블에 일치하지 않더라도 해당 행을 포함시키는 방식입니다. 

### 🧵 EQUI JOIN 

동일한 컬럼을 명시해줘서 두개의 테이블을 연결 시킨다.

```sql 
SELECT *
FROM emp, dept
WHERE emp.deptno = dept.deptno;
```

두 개의 테이블을 FROM절에서 선언하고 WHERE절에서 공통되는 컬럼을 묶어준다. 

이렇게 두 테이블을 연결하면 공통되는 컬럼이 있는 데이터만 연결되지만, 각 테이블 마다 1개씩 존재하며 나머지 컬럼들도 각자의 테이블에 존재한다.

그러므로 공통되는 컬럼을 SELECT절에서 사용하고 싶다면 테이블과 함께 명시해줘야한다. (공통되지 않는 컬럼. 즉,1개만 존재하는 컬럼은 명시하지 않아도 무방함)

### 🧵 NON EQUI JOIN 

두 테이블 간의 동등한 조건을 열을 기준으로 하지 않고, 다른 조건을 사용하여 조인을 한다. 

일반적으로 비교연산자 (>, <, >=, <=)를 사용하여 조인의 조건을 정의하며 비교조건은 일치하지 않는 값을 가진 행도 선택가능하다. 

``` 
SELECT *
FROM emp a JOIN dept b
ON a.empno > b.deptno AND b.deptno = a.deptno;
```

위와 같이 비교연산자를 사용하여 JOIN의 조건의 설정이 가능하며, ON에는 하나의 조건이 아닌 여러개의 조건이 올 수 있음을 기억하자.


### 🧵 INNER JOIN 

![INNER_JOIN](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/7b474c55-d439-41ca-bcad-9a672848061e)

오른쪽 테이블에 일치하는 레코드가 있는 모든 왼쪽 레코드를 반환해줍니다.

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

### 🧵 NATURAL JOIN

```sql 
SELECT deptno, ename, dname, loc
FROM emp a NATURAL JOIN dept b;
```

NATURAL JOIN의 경우에는 사용자가 어떤 공통된 컬럼을 사용할지 설정하는 것이 아닌 공통된 컬럼명이 공통의 내용을 가진다면 알아서 인식하여 그 내용만 사용하겠다는 의미이다.

이때, 공통컬럼이 하나가 아니라 여러개여도 그 모두가 합쳐지며 USING과 마찬가지로 공통된 컬럼은 하나의 컬럼으로 합쳐져 테이블을 명시하여 사용이 불가능하다.

### 🧵 OUTER JOIN

OUTER JOIN은 데이터의 공통된 데이터 상관없이 묶어줄 수 있기 때문에 합집합의 개념으로 생각하고 살펴보는 것이 좋아보입니다.

OUTER JOIN의 경우는 emp, dept 테이블은 이해하기에 그렇게 좋은 데이터는 아닌듯 하여 새로운 데이터를 사용하여 설명하겠습니다.

실제 데이터를 삽입해보면서 확인해 보시며 이해하는 것을 추천합니다. 

<p><details>
<summary style="color:red;">테스트를 위한 테이블 생성 코드</summary><!-- summary 아래 한칸 공백 필요 -->

<pre><code class="language-sql">
```
-- 해당 코드는 참고1 페이지에서 사용한 데이터임
-- TABLE_A 생성
CREATE TABLE TABLE_A (
  id INT,
  name VARCHAR(255),
  PRIMARY KEY (id)
);

-- TABLE_B 생성
CREATE TABLE TABLE_B (
  id INT,
  name VARCHAR(255),
  PRIMARY KEY (id)
);

-- TABLE_A 데이터 삽입
INSERT INTO TABLE_A (id, name)
VALUES (1, 'FOX');
INSERT INTO TABLE_A (id, name)
VALUES (2, 'COP');
INSERT INTO TABLE_A (id, name)
VALUES (3, 'TAXI');
INSERT INTO TABLE_A (id, name)
VALUES (6, 'WASHINGTON');
INSERT INTO TABLE_A (id, name)
VALUES (7, 'DELL');
INSERT INTO TABLE_A (id, name)
VALUES (5, 'ARIZONA');
INSERT INTO TABLE_A (id, name)
VALUES (4, 'LINCOLN');
INSERT INTO TABLE_A (id, name)
VALUES (10, 'LUCENT');
  
  
-- TABLE_B 데이터 삽입
INSERT INTO TABLE_B (id, name)
VALUES (1, 'TROT');
INSERT INTO TABLE_B (id, name)
VALUES (2, 'CAR');
INSERT INTO TABLE_B (id, name)
VALUES (3, 'CAB');
INSERT INTO TABLE_B (id, name)
VALUES (6, 'MONUMENT');
INSERT INTO TABLE_B (id, name)
VALUES (7, 'PC');
INSERT INTO TABLE_B (id, name)
VALUES (8, 'MICROSOFT');
INSERT INTO TABLE_B (id, name)
VALUES (9, 'APPLE');
INSERT INTO TABLE_B (id, name)
VALUES (11, 'SCOTCH');
```
</code></pre>
<hr/>

</details></p>

#### 🔗 LEFT OUTER JOIN

![LEFT_JOIN](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/7bc65578-dbb3-4a3c-8807-d0769365279f)

왼쪽 테이블의 모든 레코드를 출력하며, 오른쪽 테이블에 일치하는 레코드가 있는 경우 해당 레코드도 같이 반환합니다.

LEFT OUTER JOIN은 OUTER을 생략해도 동일한 코드이다.

```sql 
-- LEFT JOIN
SELECT A.PK AS A_PK, A.Value AS A_Value,
B.Value AS B_Value, B.PK AS B_PK
FROM Table_A A
LEFT JOIN Table_B B
ON A.PK = B.PK

-- LEFT를 명시하지 않고도 가능하다. (+) 사용 +가 명시된쪽 데이터는 전부 출력
SELECT A.PK AS A_PK, A.Value AS A_Value,
B.Value AS B_Value, B.PK AS B_PK
FROM Table_A A
JOIN Table_B B
ON A.PK(+) = B.PK
```

| A_PK  | A_Value | B_Value | B_PK |
|----|-----|-----|-----|
|1| FOX    |   TROT    |      1|
|2| COP     |   CAR     |      2|
|3|TAXI      | CAB      |     3|
|4| LINCOLN  |  NULL    |   NULL|
|5|ARIZONA   | NULL     |  NULL|
|6| WASHINGTON| MONUMENT |     6|
|7| DELL      | PC        |    7|
|10| LUCENT   |  NULL     |  NULL|


#### 🔗 RIGHT OUTER JOIN

![RIGHT_JOIN](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/9b23b1d6-79be-4ab5-817b-5ffc38bd4bff)

오른쪽 테이블의 모든 레코드를 출력하며, 왼쪽 테이블에 일치하는 레코드가 있는 경우 해당 레코드도 같이 반환합니다.

RIGHT OUTER JOIN은 OUTER을 생략해도 동일한 코드이다.

```sql 
-- RIGHT JOIN
SELECT A.id AS A_PK, A.name AS A_Value,
B.name AS B_Value, B.id AS B_PK
FROM Table_A A
RIGHT JOIN Table_B B
ON A.id = B.id;

-- LEFT를 명시하지 않고도 가능하다. (+) 사용 +가 명시된쪽 데이터는 전부 출력
SELECT A.PK AS A_PK, A.Value AS A_Value,
B.Value AS B_Value, B.PK AS B_PK
FROM Table_A A
JOIN Table_B B
ON A.PK = B.PK(+)
```

| A_PK      | A_Value    | B_Value   | B_PK |
|-----------|------------|-----------|-----|
| 1         | FOX        | TROT      |      1|
| 2         | COP        | CAR       |      2|
| 3         | TAXI       | CAB       |     3|
| 6         | WASHINGTON |  MONUMENT |     6|
| 7         | DELL       | PC         |  7   |
 | NULL| NULL     | MICROSOFT  | 8       |
| NULL| NULL      | APPLE      | 9         |
 | NULL|  NULL    | SCOTCH     | 11       |

#### 🔗 FULL OUTER JOIN

![FULL_OUTER_JOIN](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/e8ac414a-f90b-4dc3-8317-b8a0eb44136f)

FULL OUTER JOIN은 오른쪽 테이블의 레코드와 왼쪽 레코드를 결합하여 모든 레코드를 반환합니다.

LEFT JOIN과 RIGHT JOIN이 결합이며, 일치하는 행, 일치하지 않는 행 모두를 반환한다. 즉, 양측에서 한 테이블에서만 존재하는 행도 전부 가져온다.

```sql 
-- OUTER JOIN
SELECT A.id AS A_PK, A.name AS A_Value,
B.name AS B_Value, B.id AS B_PK
FROM Table_A A
FULL OUTER JOIN Table_B B
ON A.id = B.id;
```

| A_PK      | A_Value    | B_Value  | B_PK |
|-----------|------------|----------|-----|
|1| FOX        | TROT     | 1       |
|2| COP        | CAR      |       2 |
|3 | TAXI       | CAB      |    3
|6 | WASHINGTON | MONUMENT |   6     |
|7 | DELL       | PC       |      7
|NULL|  NULL      | MICROSOFT  |    8     |
|NULL | NULL       | APPLE      |     9|
|NULL | NULL       | SCOTCH     |    11|
|5 | ARIZONA    | NULL       |   NULL|
|4 | LINCOLN    | NULL       |    NULL|
|10 | LUCENT     |  NULL      |   NULL|


#### 🔗 LEFT Excluding JOIN

![LEFT_EXCLUDING_JOIN](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/f8a2a85c-1fa3-4325-be23-a8020afad27c)

LEFT OUTER JOIN에서 공통된 부분을 제외한 데이터, 즉 차집합의 데이터를 반환한다. 

문법이 존재한다기 보다는 OUTER JOIN의 특징을 활용하여 데이터를 추출한다. 

왼쪽 테이블을 기준으로 모든 데이터를 반환하고 공통되지 않는 데이터가 있을 때 NULL 값이 들어가게된다. 

여기서 WHERE 절에 IS NULL을 사용하여 값을 추출한다.

```sql 
-- LEFT EXCLUDING JOIN
SELECT A.id AS A_PK, A.name AS A_Value,
B.name AS B_Value, B.id AS B_PK
FROM Table_A A
LEFT JOIN Table_B B
ON A.id = B.id
WHERE B.id IS NULL;
```

| A_PK | A_Value | B_Value   | B_PK |
|------|---------|-----------|-----|
| 4    | LINCOLN | NULL    |      NULL |
| 5    | ARIZONA | NULL    | NULL    |
 | 10   | LUCENT  |   NULL  | NULL    |


#### 🔗 RIGHT Excluding JOIN

![RIGHT_EXCLUDING_JOIN](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/2ea43219-d3aa-4272-99d1-041ac60acdd2)

LEFT Excluding JOIN 와 동일하지만 기준이 오른쪽 테이블이다.

```sql 
-- RIGHT EXCLUDING JOIN
SELECT A.id AS A_PK, A.name AS A_Value,
B.name AS B_Value, B.id AS B_PK
FROM Table_A A
RIGHT JOIN Table_B B
ON A.id = B.id
WHERE A.id IS NULL;
```

| A_PK                            | A_Value                    | B_Value            | B_PK  |
|---------------------------------|----------------------------|--------------------|-------|
| NULL| NULL     | MICROSOFT   | 8     |
 | NULL | NULL    | APPLE      | 9     |
 | NULL | NULL    |    SCOTCH    |    11 |


#### 🔗 OUTER Excluding JOIN

![OUTER_EXCLUDING_JOIN](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/5f2387e8-18d1-4696-b747-b66f61eba293)

이 쿼리문은 공통의 데이터를 제외하고 왼쪽 테이블, 오른쪽 테이블의 모든 레코드를 반환한다.

```sql 
-- OUTER EXCLUDING JOIN
SELECT A.id AS A_PK, A.name AS A_Value,
B.name AS B_Value, B.id AS B_PK
FROM Table_A A
FULL OUTER JOIN Table_B B
ON A.id = B.id
WHERE A.id IS NULL
OR B.id IS NULL;
```

| A_PK                            | A_Value                    | B_Value            | B_PK  |
|---------------------------------|----------------------------|--------------------|-------|
|NULL |NULL |      MICROSOFT  |   8|
|NULL |NULL  |     APPLE     |    9|
|NULL| NULL  |     SCOTCH   |    11|
|5 |ARIZONA   | NULL       |NULL|
|4 |LINCOLN   | NULL      | NULL|
|10| LUCENT   |  NULL     |  NULL|

### 🧵 CROSS JOIN

두 개의 테이블로 조합하여 만들수 있는 모든 조합을 생성하여 반환한다.

CROSS JOIN을 명시해도 되고 따로 명시하지 않고 두 개의 테이블을 사용하면 자동으로 생성된다.

```sql 
-- CROSS JOIN (명시)
SELECT A.id AS A_PK, A.name AS A_Value,
B.name AS B_Value, B.id AS B_PK
FROM Table_A A CROSS JOIN Table_B B;

-- CROSS JOIN (명시X)
SELECT A.id AS A_PK, A.name AS A_Value,
B.name AS B_Value, B.id AS B_PK
FROM Table_A A, Table_B B;
```

### 🧵 SELF JOIN

SELF JOIN은 자기자신을 사용하여 연결하는 것이다. 실질적인 방법은 위에 설명한 내용들과 동일함.

```sql
-- a테이블에서 사원의 매니저 데이터만 출력하는 쿼리문이다. 
SELECT a.empno "사원번호", a.ename "사원명", a.mgr "관리자번호", b.ename "관리자명"
FROM emp a, emp b
WHERE a.mgr = b.empno;
```



***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🧐

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}