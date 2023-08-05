---
title:  "[DB/OracleDB] 집합"  
excerpt: "JOIN, 집합"

categories: # 분류하고싶은 카테고리 입력
  - DataBase
tags:     # 중요 키워드
  - [OracleDB]

toc: true
toc_sticky: true
published: true

date: 2023-06-18
last_modified_at: 2023-06-18
---


인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1](https://pearlluck.tistory.com/46)
{: .notice--warning}

## 집합

### 합집합(UNION)

두 개 이상의 집합을 합쳤을 때의 결과로, 합쳐진 모든 원소를 포함하는 새로운 집합을 생성합니다. 

오라클 DB에서 힙집합과 관련된 함수는 UNION, UNION ALL 이 있다. 

`UNION 연산은 두 개 이상의 쿼리 결과나 테이블을 합쳐서 중복을 제거한 결과`를 반환합니다. 중복된 행은 한 번만 포함됩니다. 즉, 중복된 결과는 제거되고 유일한 값만 반환됩니다. 

`UNION ALL 연산은 두 개 이상의 쿼리 결과나 테이블을 합쳐서 중복을 제거하지 않고 모든 결과`를 반환합니다. 중복된 행도 모두 포함됩니다.

> UNION 

```sql 
-- 1. emp 테이블에서 job이 MANAGER인 데이터
SELECT empno 
FROM emp
WHERE job = 'MANAGER';

-- 2. emp 테이블에서 sal이 2500이상인 데이터
SELECT empno
FROM emp
WHERE sal >= 2500;

-- 3. 두 테이블을 UNION 
SELECT empno 
FROM emp
WHERE job = 'MANAGER'
UNION
SELECT empno
FROM emp
WHERE sal >= 2500;
```

결과를 쉽게 확인하기 위해 empno 하나의 컬럼만 사용하여 비교하면 

1번은 7566, 7698, 7782 이 나오고, 2번은 7566, 7698, 7839, 7902 이 나온다. 

두 결과를 UNION을 통하여 합쳐준 3번은 7566, 7698, 7782, 7839, 7902 가 나오는데 1,2번의 중복값 7566, 7698은 한 번씩만 출력되는 것을 확인할 수있다.

> UNION ALL 

```sql 
-- 4. 두 테이블을 UNION ALL 
SELECT empno 
FROM emp
WHERE job = 'MANAGER'
UNION ALL
SELECT empno
FROM emp
WHERE sal >= 2500;
```

4번의 결과는 UNION ALL 통하여 합쳐주는 쿼리문이며 결과는 7566, 7698, 7782, 7566, 7698, 7839, 7902 가 출력되며 중복된 값을 제거하지 않고 모든 결과를 반환합니다.

### 교집합(INTERSECTION)

두 개 이상의 집합에서 동시에 포함되는 모든 원소로 이루어진 새로운 집합을 생성합니다.

`INTERSECT 키워드를 사용`해서 구할 수 있다. 

```sql 
-- 5. 두 테이블을 UNION ALL 
SELECT empno 
FROM emp
WHERE job = 'MANAGER'
INTERSECT
SELECT empno
FROM emp
WHERE sal >= 2500;
```

5번의 결과는 7566, 7698 이 나오며, 1번과 2번의 중복되는 값만 출력한다. 


### 차집합(DIFFERENCE)

하나의 집합에서 다른 집합에 속하지 않는 원소들로 이루어진 새로운 집합을 생성합니다.

`MINUS 키워드를 사용하여 차집합을 구할 수 있으며, 차집합은 기준이 중요`하다. 

```sql 
-- 6. 두 테이블을 MINUS ( 2 - 1 ) 
SELECT empno
FROM emp
WHERE sal >= 2500
MINUS
SELECT empno 
FROM emp
WHERE job = 'MANAGER';

-- 7. 두 테이블을 MINUS ( 1 - 2 ) 
SELECT empno 
FROM emp
WHERE job = 'MANAGER'
MINUS
SELECT empno
FROM emp
WHERE sal >= 2500;
```

6번은 7566, 7698, 7839, 7902에서 공통결과인 7566, 7698을 제외한 7839, 7902 가 출력되고,

7번은 7566, 7698, 7782에서 공통결과인 7566, 7698을 제외한 7782 가 출력된다.

차집합을 사용할 땐, 기준을 잘 생각하고 사용해야 한다. 



***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🧐

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}