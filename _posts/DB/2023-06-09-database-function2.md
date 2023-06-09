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
last_modified_at: 2023-06-09
---


인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1](https://kkkapuq.tistory.com/106)
{: .notice--warning}


## 문자관련 함수

SUBSTR, INSTR, LENGTH, LPAD, LTRIM.....

### SUBSTR

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