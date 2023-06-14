---
title:  "[DB] OracleDB 날짜관련 함수"  
excerpt: "TO_DATE"

categories: # 분류하고싶은 카테고리 입력
  - ORACLEDB
tags:     # 중요 키워드
  - [OracleDB, TO_DATE]

toc: true
toc_sticky: true

date: 2023-06-14
last_modified_at: 2023-06-14
---


인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1](https://product.kyobobook.co.kr/detail/S000001032057)
{: .notice--warning}

## 날짜 데이터 다루기

오라클 DB에서 날짜 타입은 크게 DATE 타입과 TIMESTAMP 타입으로 2가지 입니다. 

DATE 타입은 'YYYY-MM-DD HH24:MI:SS'와 같이 날짜와 시간을 포함하며, 

TIMESTAMP 타입은 'YYYY-MM-DD HH24:MI:SS.FF' 날짜와 시간과 초의 소수점까지 나타내어 DATE 보다 더 정밀한 시간 정보를 포함합니다.

- 날짜 형식의 의미
- YYYY: 연도
- MM: 월 
- DD: 일 
- HH24: 시간 (24시간 형식, 두 자리)
- MI: 분 (두 자리)
- SS: 초 (두 자리)
- FF: 소수 초 (나노초까지 표현)
- TZH: 시간대 오프셋의 시간 (시간대를 나타내는 시간 차)
- TZM: 시간대 오프셋의 분 (시간대를 나타내는 분 차)

### 날짜에서 데이터 추출

날짜에서 년,월,일,시,분,초 가 따로따로 필요한 경우가 존재한다. 

이때, EXTRACT 함수를 사용해서 원하는 데이터를 추출할 수 있다. 

```sql 
SELECT 
    EXTRACT(YEAR FROM SYSDATE) YEAR,
    EXTRACT(MONTH FROM SYSDATE) MONTH,
    EXTRACT(DAY FROM SYSDATE) DAY,
    EXTRACT(HOUR FROM CAST(SYSDATE AS TIMESTAMP)) HOUR,
    EXTRACT(MINUTE FROM CAST(SYSDATE AS TIMESTAMP)) MINUTE,
    EXTRACT(SECOND FROM CAST(SYSDATE AS TIMESTAMP)) SECOND
FROM dual;
```

년,월,일은  DATE, TIMESTAMP 어떤 타입으로 설정해도 EXTRACT 함수를 사용할 수 있지만,

시,분,초는 TIMESTAMP 타입만 가능하여 날짜 데이터가 DATE 타입이라면 TIMESTAMP 로 변환해줘야한다.

여기서 주의해야 할 점은 SYSDATE 를 타입변환하여 초를 추출했으므로 초의 정수자리만 출력되지만, 처음부터 타입이 TIMESTAMP 타입에서 추출했다면 더 큰범위인 초의 소수점까지 출력하게 됩니다.

> 타입 변환하는 방법 CAST(표현식 AS 데이터타입)
> > 표현식은 변환할 값을 나타내며, 데이터 타입은 표현식을 변환할 데이터 타입입니다.   
> > EX)  CAST(1234 AS VARCHAR2(20)) , CAST('1234' AS NUMBER), CAST(SYSDATE AS TIMESTAMP)

***


### 날짜 자르기 

날짜를 잘라주는 함수 TRUNC를 활용해서 형식을 맞추거나 날짜를 초기화 할 수 있다.  

초기화라는 의미는 TRUNC로 잘라준 기준에 따라 값이 바뀌는 것을 의미하는데 예시를 통해 이해하는게 좋을듯 하다.

```sql 
SELECT 
       SYSDATE  AS 현재시간,
       TRUNC(SYSDATE) AS 시간절사,
       TRUNC(SYSDATE, 'DD') AS 시간절사2, --TRUNC(SYSDATE)와 동일
       TRUNC(SYSDATE, 'HH24') AS 분초자르기,
       TRUNC(SYSDATE, 'MI') AS 초자르기,
       TRUNC(SYSDATE, 'YEAR') AS 월일초기화, -- TRUNC(SYSDATE, 'YYYY')와 동일
       TRUNC(SYSDATE, 'MM') AS 일초기화, --TRUNC(SYSDATE, 'MONTH')와 동일
       TRUNC(SYSDATE, 'DAY') AS 요일초기화 --해당 주의 일요일로 초기화 (일요일이 주의 시작)
FROM dual;
```

> 결과
- 현재시간 : 2023-06-14 10:22:10	
- 시간절사 : 2023-06-14 00:00:00	
- 시간절사2 : 2023-06-14 00:00:00	
- 분초자르기 : 2023-06-14 10:00:00	
- 초자르기 : 2023-06-14 10:22:00	
- 월일초기화 : 2023-01-01 00:00:00	
- 일초기화 : 2023-06-01 00:00:00	
- 요일초기화 : 2023-06-11 00:00:00


`사용중인 오라클 DB에서 시,분,초가 출력되지 않을 때`

```
-- NLS 설정 확인
SELECT * FROM NLS_SESSION_PARAMETERS;

-- 날짜 시,분,초 모두 출력되도록 설정
-- 아래 명령어로 DATE타입 형식 변경(형식은 원하는 형식을 작성해주면 된다.)
ALTER SESSION SET NLS_DATE_FORMAT = 'YYYY-MM-DD HH24:MI:SS';
```

NLS_SESSION_PARAMETERS는 ORACLE DB에서 세션의 언어 및 날짜 형식과 같은 설정을 의미한다. 

이 테이블의 정보를 수정하여 언어, 날짜 및 숫자의 형식, 통화 기호등을 지정할 수 있다.

- NLS_LANGUAGE: 세션의 언어를 지정합니다.
- NLS_DATE_FORMAT: 날짜 표시 형식을 지정합니다.
- NLS_NUMERIC_CHARACTERS: 숫자 형식에서 사용되는 기호를 지정합니다.
- NLS_CURRENCY: 통화 기호를 지정합니다.
- 기타 등등......


### 날짜 더하기

```sql 
SELECT 
    SYSDATE AS 현재시간,
    (ADD_MONTHS(SYSDATE, 1)) AS "1달후",
    (SYSDATE+1) AS "1일후",
    (SYSDATE+(1/24)) AS "1시간후",
    (SYSDATE+(1/24/60)) AS "1분후",
    (SYSDATE+(1/24/60/60)) AS "1초후"
FROM dual;
```

- 현재시간 : 2023-06-14 10:55:05	
- 1달후 : 2023-07-14 10:55:05	
- 1일후 : 2023-06-15 10:55:05	
- 1시간후 : 2023-06-14 11:55:05	
- 1분후 : 2023-06-14 10:56:05	
- 1초후 : 2023-06-14 10:55:06

> 만약 시,분,초 가 안나온다면 NLS_DATE_FORMAT을 변경해주거나, TO_CHAR로 형식을 지정하여 출력해 확인해줘야 한다.

월단위를 연산하고 싶으면 ADD_MONTHS 함수 사용 (음수 설정 가능)


### TO_DATE (문자 -> 날짜) 

TO_DATE 함수는 날짜형식으로 작성되어 있는 문자를 DATE 타입으로 변경해준다.

```sql 
SELECT TO_DATE('2023-06-14', 'YYYY-MM-DD') AS 변환된날짜
FROM DUAL;
```

- 변환된날짜 : 2023-06-14

```sql 
SELECT TO_DATE('2023-06-14 10:30:00', 'YYYY-MM-DD HH24:MI:SS') AS 변환된날짜시간
FROM DUAL;
```

- 변환된날짜시간 : 2023-06-14 10:30:00

바뀐게 없어보일수 있지만, 형식을 달리해주거나 비교하거나 날짜와 관련된 함수를 사용하기 위해서 타입 변환이 필요할 때 사용한다.


### 기타등등..

#### MONTHS_BETWEEN : 날짜와 날짜 사이의 기간을 월단위로 차이를 계산한 값

MONTHS_BETWEEN(날짜1, 날짜2) 형식으로 작성하며 '날짜1 - 날짜2' 로 연산되기 때문에 음수값에 주의해야한다.

```sql 
SELECT MONTHS_BETWEEN(SYSDATE, '2023/12/31') 차이값
FROM dual;
```

- 차이값 : -6.55

2023년 12월 31일에서 오늘 날짜를 뺸것이 아닌 오늘 날짜에서 2023년 12월 31일 을 뺀값이라 음수가 나온다.

#### LAST_DAY

입력된 날짜의 해당 월 마지막날을 출력한다. (시,분,초가 함께 입력되어있느면 시,분,초는 변하지 않음)

```sql 
SELECT LAST_DAY(SYSDATE) 이번달마지막날
FROM dual;
```

- 이번달마지막날 : 2023-06-30 11:07:57


#### NEXT_DAY

NEXT_DAY(날짜, 요일) 형식으로 작성하며 입력된 날짜 이후로 부터 입력한 요일을 만나는 가장 첫번째 날을 출력함

```sql 
SELECT NEXT_DAY(SYSDATE, '금요일') 다음금요일
FROM dual;
```

- 다음금요일 : 2023-06-16 11:09:31



***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🧐

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}