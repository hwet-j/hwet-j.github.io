---
title:  "[DB] OracleDB 기타 함수"  
excerpt: "NULL, CASE"

categories: # 분류하고싶은 카테고리 입력
  - ORACLEDB
tags:     # 중요 키워드
  - [OracleDB, CASE, NULL]

toc: true
toc_sticky: true

date: 2023-06-17
last_modified_at: 2023-06-17
---


인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1](https://product.kyobobook.co.kr/detail/S000001032057)
{: .notice--warning}


## 여러가지 함수

각 함수를 설명하기 위해서 Scott 계정에 기본적인 EMP 테이블을 사용하여 예시를 들겠습니다.

[EMP테이블](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/bd1b5b9e-26ff-4049-918c-20e6d0ef9d08)

### NULL값 다루기

데이터를 다루다보면 명시적으로 또는 암시적으로 NULL 데이터가 들어가 있는 경우가 있는데 이 NULL을 다룰줄 알아야 원하는 대로 데이터를 다룰 수 있다.

#### 👉 IS NULL / IS NOT NULL

IS NULL은 데이터가 NULL 인가 확인하는 명령어

IS NOT NULL은 데이터가 NULL이 아닌가 확인하는 명령어 이다.

> IS NULL

```sql 
-- emp 테이블에서 comm 컬럼의 값이 NULL인 경우만 출력
SELECT *
FROM emp
WHERE comm IS NULL;
```

`결과`

COMM 컬럼의 값이 NULL 데이터만 추출된 것을 확인할 수 있다.

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/5955a258-5cbb-45fa-9093-9c1bd6797443)

> IS NOT NULL

```sql 
-- emp 테이블에서 comm 컬럼의 값이 NULL이 아닌 경우만 출력
SELECT *
FROM emp
WHERE comm IS NOT NULL;
```

`결과`

COMM 컬럼의 값이 NULL이 아닌 데이터만 추출된 것을 확인할 수 있다.

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/3d2c0b6a-5f51-4c3a-b4d5-1759119ad8bc)

#### 👉 NVL / NVL2

두 함수 모두 NULL 값을 대체 하는 함수지만, 사용법의 차이가 있다. 

NVL 함수는 두 개의 인자가 존재하며 첫 번째 인자로 절달된 값이 NULL인 경우, 두 번째 인자로 전달된 값으로 대체합니다.

NVL2 함수는 세 개의 인자가 존재하며 첫 번쨰 인자로 전달된 값이 NULL인 경우, 세 번쨰 인자로 전달된 값으로 대체하고, NULL인 경우 두 번째 인자로 전달된 값으로 대체합니다. 

> NVL

```sql 
-- comm 컬럼의 값이 NULL이면 0으로 대체
SELECT empno, ename, NVL(comm, 0) 
FROM emp;
```

> NVL2

```sql 
SELECT empno, ename, NVL2(comm, '정보입력됨', '정보입력안됨') 
FROM emp;
```

❗ 여기서 주의 해야할 점은 데이터를 대체하는 과정에서 데이터의 타입이 컬럼의 타입에 맞게 설정해야한다.  

```sql
SELECT NVL(comm, '보너스없음') FROM emp; 
```  

위의 쿼리문을 작성하면 유효한 숫자를 입력하라는 문구가 나오며 실행되지 않는다. 타입이 일치하지 않아 발생하는 문제이다.

NVL의 경우에는 NULL이 아닌 데이터는 변화하지 않으므로 입력되는 첫 번째 인자에 맞게 타입을 설정해주면된다. 

NVL2의 경우에는 좀 더 복잡하다. 두,세 번째의 타입을 동일하게 만들어 주거나, NULL이 아닐 때의 바뀌는 데이터인 두 번째 인자에 맞춰 타입을 설정해야 한다. 

굳이 깊게 들어갈 필요는 없지만, 깊게 들어가면 오라클에서 자동형변환이 제공되는데 '문자 -> 숫자' 로 자동형변환되려면 문자의 형태가 숫자의 형태로만 이루어져 있을 때 가능하다.

'숫자 -> 문자'의 경우에는 모든 숫자는 문자로 변형 가능하기 때문에 자동형변환 된다. 

```sql
-- NVL 
-- 에러 O / NULL -> 문자, NULL X -> comm의 타입 (문자 != 숫자) 
SELECT NVL(comm, '보너스 없음') FROM emp;

-- NVL2 (기준은 NULL X의 타입) 
-- 에러 O / NULL X -> 숫자, NULL -> 문자 
SELECT NVL2(comm, 1000, '보너스 없음') FROM emp;

-- 에러 X / NULL X -> 숫자, NULL -> 문자 (숫자형태문자 -> 숫자 변환가능)
SELECT NVL2(comm, 1000, '10') FROM emp; 

-- 에러 X / NULL X -> 숫자, NULL -> 숫자
SELECT NVL2(comm, 1000, 10) FROM emp; 

-- 에러 X / NULL X -> 문자, NULL -> 숫자 (숫자 -> 문자 변환가능)
SELECT NVL2(comm, '보너스 있음', 0) FROM emp; 

-- 에러 X / NULL X -> 문자, NULL -> 문자 
SELECT NVL2(comm, '보너스 있음', '보너스 없음') FROM emp; 
```  

문자, 숫자 타입의 경우에는 이런 형식이 가능하지만, 다른 DATE나 TIMESTAMP 등의 경우는 어떻게 다른지는 모르겠다. 

확실한건 출력되는 타입을 맞춰줘야지만 에러가 발생하지 않는다는 것이니 이점을 주의해서 사용하자.

#### 👉 NULLIF 

NULLIF는 두 개의 인자를 필요로 하며, 두 인자값을 비교하여 동일한 경우 NULL을 반환하고, 다른경우에는 첫 번째 인자를 반환합니다. 

```sql 
-- comm 컬럼의 값이 0이면 null 값으로 대체
SELECT NULLIF(comm, 0)
FROM emp;
```

#### 👉 COALESCE

COALESCE는 인자를 최소 두 개만 작성해주면 되며 최대 인자수는 따로 존재하지 않는것 같다.

인자를 작성한 순서대로 데이터를 확인하며 데이터가 NULL인지 확인하며, NULL이 아닌 값이 나오면 값을 반환하고 다음 record 출력으로 넘어간다.  

```sql 
-- emp 테이블에서 comm , mgr중에서 NULL이 아닌 첫 번째 값을 반환
SELECT COALESCE(comm , mgr) FROM emp;
```

❗ 여기서도 나열되는 타입이 동일하게 설정하여 하나의 컬럼에서 나오는 데이터가 하나의 데이터 타입으로 출력가능 하도록 설정해야 한다. 

```sql 
-- emp 테이블에서 comm , mgr, job중에서 NULL이 아닌 첫 번째 값을 반환
SELECT COALESCE(comm , mgr, job) FROM emp;
```  

위의 쿼리문은 실행할 수 없는 쿼리문이다. 

comm, mgr은 숫자(NUMBER) 타입이지만, job의 경우에는 문자(VARCHAR2)타입이기 때문에 데이터 타입이 맞지 않기 때문이다. 

### 조건에 따라 값 변환하기

NULL을 다루면서 NULL 값인가 아닌가에 따라 값을 변환하는 함수를 몇가지 다뤘는데, NULL 이외에 다른 조건에 따라 값을 변환하는 함수를 알아보자.

#### 👉 DECODE

DECODE의 기본적인 문법 형태는 `DECODE(expression, value1, result1, value2, result2, ..., default_result)` 의 형태이다.

DECODE는 가장 첫 인자에 표현식(expression)과 value가 동일하면, 해당 value에 맞는 result 값을 반환한다. 

DECODE의 경우에는 코드를 좀 더 이해하기 쉽도록 scott 계정에 존재하지 않는 데이터로 예를 들겠습니다.

```sql 
-- students 테이블에서 score 컬럼을 기반으로 성적 등급 변환
SELECT student_name, DECODE(score,
                           90, 'A',
                           80, 'B',
                           70, 'C',
                           'D') AS grade
FROM students;
```

학생의 점수에 따라 학생의 등급을 나눠 등급을 출력하도록 하는 쿼리문입니다.

하지만 DECODE의 경우에는 특정 값을 기준으로 조건을 설정하여 결과를 반환하기 때문에 조건식을 사용하여 결과를 반환할 수 없습니다.

위의 쿼리문을 가지고 설명을 하면 90점일 때 A, 80점일 떄 B, 70점 일때 C, 이외는 D 로 등급을 나누는데 조건(value)로 설정된 값과 동일해야지만 그에 대응하는 결과 값을 반환한다.

만약 85점, 71점 과 같은 데이터가 존재하면 B, C 등급이 아닌 이외의 데이터(default_result)인 D등급이 되어버린다. 

만약 조건식을 활욯아여 값을 조건에 따라 바꾸고 싶다면, CASE 문을 사용하면된다.

#### 👉 CASE

CASE문은 WHEN, THEN , ELSE , END와 함께 사용된다. 일반적인 IF, ELSE IF 문으로 생각하면 이해가 쉬울거 같다. 

WHEN 뒤에 논리식이나 비교식의 표현이 가능하고 THEN은 조건이 참인 경우에 해당하는 결과를 반환하는 값이다. ELSE는 생략가능하고 CASE문의 마지막에 END를 명시해준다.

> 기본형 (조건식) 

```sql 
CASE
    WHEN condition1 THEN result1
    WHEN condition2 THEN result2
    ...
    ELSE result
END
```

condition 위치에 조건식이 작성하고, result위치에 반환값을 작성한다. ELSE는 어떤 조건도 만족하지 않으면 반환할 값이다.

CASE문은 DECADE문과 의미가 완전히 동일하게 작성가능한데, CASE문 뒤에 표현식을 작성하면 WHEN 절에는 조건식이 아닌 특정값이 들어간다.

```sql 
CASE expression
    WHEN value1 THEN result1
    WHEN value2 THEN result2
    ...
    ELSE result
END
```

> CASE문 예시  

```sql 
-- 1. students 테이블에서 조건식에 따라 등급을 반환 
SELECT student_name,
       CASE
           WHEN score >= 90 THEN 'A'
           WHEN score >= 80 THEN 'B'
           WHEN score >= 70 THEN 'C'
           ELSE 'D'
       END AS grade
FROM students;

-- 2. DECADE문에서 작성한 데이터와 동일한 의미를 가진 CASE 문
SELECT student_name,
       CASE score
           WHEN 90 THEN 'A'
           WHEN 80 THEN 'B'
           WHEN 70 THEN 'C'
           ELSE 'D'
       END AS grade
FROM students;
```

예시의 1번 쿼리문의 경우에는 하나의 표현식(컬럼)을 사용하지않고 여러개의 다양한 데이터에서 식을 설정해도 상관은 없다.


***

***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🧐

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}