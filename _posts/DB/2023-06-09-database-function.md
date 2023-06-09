---
title:  "[DB] OracleDB 숫자관련 함수"  
excerpt: "COUNT, MAX, AVG, ROUND, TRUNC 숫자관련 함수다루기"

categories: # 분류하고싶은 카테고리 입력
  - ORACLEDB
tags:     # 중요 키워드
  - [OracleDB, TRUNC]

toc: true
toc_sticky: true

date: 2023-06-09
last_modified_at: 2023-06-09
---


인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1](https://kkkapuq.tistory.com/106)
{: .notice--warning}


## 숫자관련 함수

COUNT, MAX, MIN, AVG, ABS, ROUND, CEIL, FLOOR, TRUNC....

***

### 📌 COUNT

COUNT(컬럼) : 해당 컬럼의 null값을 제외한 ROW수

COUNT(*) : 해당 테이블에서 나오는 모든 ROW수(null포함)

> Scott 계정의 emp 테이블의 comm 컬럼의 데이터 

| INDEX | COMM |
|-------|------|
| 1     | 0    |
| 2     |      |
| 3     | 300  |
| 4     | 500  |
| 5     |      |
| 6     | 500  |
| 7     |      |
| 8     |      |
| 9     |      |
| 10    | 0    |
| 11    |      |
| 12    |      |
| 13    |      |

```sql 
-- EMP 테이블의 COMM 컬럼으로 예시이며 COMM은 총 13개의 데이터가 있으며, null값이 8개인 데이터이다.

-- emp 테이블의 전체 ROW 수 - 13개 (null값 포함)
SELECT COUNT(*) FROM emp;

-- comm이 null인 데이터의 개수 - 8개
SELECT COUNT(*) FROM emp
WHERE comm is null;

-- emp 테이블의 comm의 개수 - 5개 (null값제외)
SELECT COUNT(comm) FROM emp;
```

### 📌 MAX, MIN 

MAX(컬럼) : 출력되는 데이터중에 최대값을 리턴해준다.(null 제외)

MAX(컬럼) : 출력되는 데이터중에 최소값을 리턴해준다.(null 제외)

```sql 
-- comm의 최대값과 최소값 출력 - 최대 500, 최소 0
SELECT MAX(comm) 최대, MIN(comm) 최소
FROM emp;
```

### 📌 SUM, AVG

SUM : 출력되는 데이터들의 합을 리턴해준다.(null값제외)

AVG : 출력되는 데이터들의 평균값을 리턴해준다.(null값제외)

```sql 
-- comm의 평균과 합을 출력 - 평균 260, 최소 1300 
SELECT AVG(comm) 평균, SUM(comm) 합
FROM emp;
```

### 📌 ABS, ROUND, CEIL, FLOOR 

ABS : 입력된 데이터의 절대값을 리턴해준다.

ROUND : 입력된 데이터를 반올림 해준다.

CEIL : 입력된 데이터를 올림 해준다.

FLOOR : 입력된 데이터를 내림 해준다.

```sql 
-- ABS : 절대값처리, 최소에서 최대를 빼주면 음수지만 ABS 함수를통해 양수출력
SELECT ABS(MIN(comm)-MAX(comm)) 절대값처리
FROM emp;

-- ROUND : 반올림, 작성해준 소수점자리까지만 출력된다. 설정 소수점 바로아래에서 반올림작업
SELECT ROUND(20125.1235486,3) 소수점3, ROUND(20125.1235486,4) 소수점4, ROUND(20125.1235486,5) 소수점5
FROM dual;

-- 음수 설정도 가능하다. 음수로 설정하면 소수점이아닌 정수의 자리수를 의미함
SELECT ROUND(20125.1235486,-1) 일의자리, ROUND(20125.1235486,-2) 십의자리, ROUND(20125.1235486,-4) 백의자리
FROM dual;

-- CEIL : 올림, 정수값을 반환하며 입력매개변수는 하나이다. 소수점아래 정보는 버리면서 +1한다고 보면된다.
SELECT CEIL(20125.123) 올림1, CEIL(20125.9) 올림2
FROM dual;

-- FLOOR : 내림, 정수값을 반환하며 입력매개변수는 하나이다. 소수점아래 정보를 버리고 정수값만 가져온다.
SELECT FLOOR(20125.123) 내림1, FLOOR(20125.9) 내림2
FROM dual;
```

> 절대값처리  : 500  
> 소수점3 : 20125.124, 소수점4 : 20125.1235, 소수점5 : 20125.12355   
> 일의자리 : 20130, 십의자리 : 20100, 백의자리 : 20000  
> 올림1 : 20126, 올림2 : 20126   
> 내림1 : 20125, 내림2 : 20125  

### 📌 TRUNC

TRUNC : 숫자나 날짜를 자를 때 사용하는 함수이다.


`TRUNC로 숫자 다루기`

```sql 
-- 숫자 자르기
SELECT 1234.567 AS "주어진수",  
TRUNC(1234.567) AS "소수점자르기",  
TRUNC(1234.567, '1') AS "소수점첫째자리자르기",  
TRUNC(1234.567, '2') AS "소수점둘째자리자르기",
TRUNC(1234.567, '-1') AS "1단위자르기",
TRUNC(1234.567, '-2') AS "10단위자르기",
TRUNC(1234.567, '-3') AS "100단위자르기"
FROM DUAL;
```

> 주어진수 : 1234.567    
> 소수점자르기 : 1234  
> 소수점첫째자리자르기 : 1234.5    
> 소수점둘째자리자르기 : 1234.56    
> 1단위자르기 : 1230    
> 10단위자르기 : 1200    
> 100단위자르기 : 1000  


`TRUNC로 날짜 다루기`

```sql 
-- 시간 자르기
SELECT SYSDATE  AS 현재시간,
       TRUNC(SYSDATE) AS 시간자르기,
       TRUNC(SYSDATE, 'DD') AS 시간자르기2, -- = TRUNC(SYSDATE)
       TRUNC(SYSDATE, 'HH24') AS 분초자르기,
       TRUNC(SYSDATE, 'MI') AS 초자르기,
       TRUNC(SYSDATE, 'YEAR') AS 월일초기화, --  = TRUNC(SYSDATE, 'YYYY')
       TRUNC(SYSDATE, 'MM') AS 일초기화, --  = TRUNC(SYSDATE, 'MONTH')
       TRUNC(SYSDATE, 'DAY') AS 요일초기화 -- 해당 주의 시작일로 초기화 (일요일이 주의 시작)
FROM DUAL;
```

> 현재시간 = 2023-06-08 오후 7:12:34  
> 시간자르기 = 2023-06-08  
> 시간자르기2 = 2023-06-08  
> 분초자르기 = 2023-06-08 오후 7:00:00  
> 초자르기 = 2023-06-08 오후 7:12:00  
> 월일초기화 = 2023-01-01  
> 일초기화 = 2023-06-01  
> 요일초기화 = 2023-06-04  



***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}