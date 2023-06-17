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

#### IS NULL / IS NOT NULL : 데이터가 NULL인가, 아닌가를 판별하는 명령어이다.

IS NULL을 WHERE 절에 사용하여서 NULL 값인 데이터만 사용한다거나 IS NOT NULL 을 사용하여 NULL 값이 아닌 데이터만 사용할 수 있다.

```sql 
-- IS NULL
SELECT *
FROM emp
WHERE comm IS NULL;
```

> 결과

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/5955a258-5cbb-45fa-9093-9c1bd6797443)

```sql 
-- IS NOT NULL
SELECT *
FROM emp
WHERE comm IS NOT NULL;
```

> 결과

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/3d2c0b6a-5f51-4c3a-b4d5-1759119ad8bc)

- comm(보너스) 컬럼의 값이 NULL 값인 데이터만 사용한다는 의미이다.

***











***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🧐

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}