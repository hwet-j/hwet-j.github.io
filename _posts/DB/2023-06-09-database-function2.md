---
title:  "[DB] OracleDB 문자관련 함수"  
excerpt: "SUBSTR, INSTR...문자관련 함수다루기"

categories: # 분류하고싶은 카테고리 입력
  - ORACLEDB
tags:     # 중요 키워드
  - [OracleDB, SUBSTR, INSTR]

toc: true
toc_sticky: true

date: 2023-06-09
last_modified_at: 2023-06-13
---


인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1](https://kkkapuq.tistory.com/106)
{: .notice--warning}


## 문자관련 함수

SUBSTR, INSTR, LENGTH, LPAD, LTRIM.....

### 문자열 변환

#### INICAP : 가장앞글자는 대문자, 나머지는 소문자로 변경 

```sql 
SELECT INITCAP('enGliSh')
FROM dual;
```

> `English` 출력


#### LOWER : 소문자로 변경

```sql 
SELECT LOWER('enGliSh')
FROM dual;
```

> `english` 출력


#### UPPER : 대문자로 변경

```sql 
SELECT UPPER('enGliSh')
FROM dual;
```

> `ENGLISH` 출력

### 문자열 합치기 

#### CONCAT : 두 개의 데이터를 연결 시켜준다.

```sql 
SELECT CONCAT('문자', '합치기')
FROM dual;
```

> `문자합치기` 출력

두 개 이상의 CONCAT을 사용하고 싶다면 CONCAT안에 CONCAT을 입력해준다.

```sql 
SELECT CONCAT(CONCAT('문자', '합치기'), '하나더')
FROM dual;
```

> `문자합치기하나더` 출력

#### || : 좌, 우측에 있는 데이터를 연결 시켜준다. 

3개 이상의 데이터를 연결할 때 CONCAT 보다 쉽게 사용 가능하다.

```sql 
SELECT '하나의'||'데이터로'||'인식하여'||'칼럼이하나'
FROM dual;
```

> `하나의데이터로인식하여칼럼이하나` 출력

#### CONCAT, || 의 혼합 사용

```sql
SELECT CONCAT('문자', '합치기') || '하나더'
FROM dual;

SELECT CONCAT('문자', '합치기' || '하나더')
FROM dual;
```

> 두 쿼리문의 결과는 `문자합치기하나더` 로 동일하다.

CONCAT 함수를 사용한 데이터와 || 로 연결가능하고, CONCAT 내부에서 || 연산자 사용도 가능하다.





SUBSTR : 문자열의 원하는 부분을 추출할 수 있는 함수이다.

```sql 
-- 매개변수 1개 (데이터 제외 매개변수)
-- 양수를 입력하면 앞에서부터 시작점을 설정한다. 시작점부터 끝까지
SELECT SUBSTR('Oracle Database', 3)
FROM dual;
-- 음수를 입력하면 뒤에서부터 시작점을 설정한다. 시작점부터 끝까지
SELECT SUBSTR('Oracle Database', -4)
FROM dual;
```

> 매개변수 1개는 입력해준 시작점 부터 끝까지 출력한다.



***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}