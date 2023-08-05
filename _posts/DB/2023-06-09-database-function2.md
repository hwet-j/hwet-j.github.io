---
title:  "[DB/OracleDB] 문자관련 함수"  
excerpt: "SUBSTR, INSTR, TO_CHAR 문자관련 함수다루기"

categories: # 분류하고싶은 카테고리 입력
  - DataBase
tags:     # 중요 키워드
  - [OracleDB, SUBSTR, INSTR, TO_CHAR]

toc: true
toc_sticky: true

date: 2023-06-09
last_modified_at: 2023-06-14
---


인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1](https://kkkapuq.tistory.com/106)
[참고2](https://product.kyobobook.co.kr/detail/S000001032057)
{: .notice--warning}


## 문자관련 함수

### 문자열 다루기

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



#### SUBSTR : 문자열의 원하는 부분을 추출할 수 있는 함수이다.

SUBSTR(STRING, START_POSITION, LENGTH) - LENGTH는 생략가능하며 생략시 문자열 끝까지 잘라준다.

```sql 
-- 매개변수 1개
-- 양수를 입력하면 앞에서부터 시작점을 설정한다. 시작점부터 끝까지
SELECT SUBSTR('Oracle Database', 3)
FROM dual;
-- 음수를 입력하면 뒤에서부터 시작점을 설정한다. 시작점부터 끝까지
SELECT SUBSTR('Oracle Database', -4)
FROM dual;
```

> 매개변수 1개는 입력해준 시작점부터 끝까지 데이터를 가져와 출력한다.

```sql 
-- 매개변수 2개
-- 잘라내고 싶은 시작점과 길이를 설정하여 잘라낸다.
SELECT SUBSTR('Oracle Database', 4, 5)
FROM dual;

SELECT SUBSTR('Oracle Database', -3, 2)
FROM dual;
```

> 시작점을 설정하고 설정한 길이만큼 출력된다. 
> > SUBSTR('Oracle Database', 4, 4) --> cle D  
> > SUBSTR('Oracle Database',-3, 2) --> as    


#### INSTR : 문자열에서 원하는 문자를 찾되 탐색의 위치와 몇번째로 나온 정보를 찾을것인가 설정하여 찾는다.

INSTR(STRING, SUBSTRING, START_POSITION, OCCUR)

INSTR은 찾아낸 정보의 첫 INDEX값을 RETURN 해준다. 

```sql
SELECT INSTR('ORACLE DATABASE', 'DATA', 5)
FROM dual;
```

> `8`을 출력함 - INDEX 5부터 탐색을 시작하며 DATA의 첫 INDEX는 8임


```sql 
SELECT INSTR('ORACLE DATABASE', 'E', 5,1), INSTR('ORACLE DATABASE', 'E', 5,2)
FROM dual;
```

> `6`과 `15`를 출력함 - INDEX 5부터 탐색을 시작하며 E의 첫 INDEX는 6이며, 2번째는 15임


#### LTRIM, RTRIM : 좌측 또는 우측에서 부터 시작하여 해당문자가 존재하면 잘라내 버린다.

```sql 
SELECT LTRIM('ORACLE DATABASE', 'O'), LTRIM('ORACLE DATABASE', 'E'),
RTRIM('ORACLE DATABASE', 'O'), RTRIM('ORACLE DATABASE', 'E')
FROM dual;
```

> `RACLE DATABASE`, `ORACLE DATABASE`, `ORACLE DATABASE`, `ORACLE DATABAS`  
> > 2,3번째 데이터가 그대로 나오는 이유는 좌측, 우측에서 시작해서 처음 만난데이터가 설정해준 문자가 아니기 때문이다.

#### TRIM : 양측에 존재하는 공백(스페이스바)를 제거해준다.

고정길이객체인 CHAR 타입이나 다른 데이터를 추출, 복사, 변환, 비교 등을 작업할때 존재하고 생기는 공백을 제거하기 위해 사용한다.

```sql 
SELECT TRIM('   ORACLE DATABASE    '), TRIM(' ORACLE DATABASE')
FROM dual;
```

> `ORACLE DATABASE`, `ORACLE DATABASE` ==> 가운데 공백은 사라지지 않는다. 

#### LENGTH : 문자열의 길이를 파악하는 함수

```sql
SELECT LENGTH('일이삼사오육'), LENGTH('abcd')
FROM dual;
```

> `6`, `4` 의 결과가 나온다. ==> 문자의 개수를 파악하는 함수로 보면 된다.(공백도 포함한다.)

#### LPAD, RPAD : 설정한 길이만큼 출력되며, 출력되는 길이를 데이터 내에서 뽑아낼 수 없다면 설정한 데이터로 채워 넣는다. 

LPAD는 데이터가 더이상없다면 좌측에 채워주고, RPAD는 우측에 채워준다. (공백도 가능)

문자의 길이를 맞추거나, 좌측정렬, 우측정렬과 같은 형식으로 출력하고 싶을 때 사용한다. 

```sql 
SELECT LPAD('ORACLE ', 10, 'DB'), RPAD('DATABASE', 15, 'ORA')
FROM dual;
```

> `DBDORACLE`, `DATABASEORAORAO` ==> 문자의 길이는 설정해준값을 지키며 빈공간에 설정한 문자의 앞에서부터 하나씩 채워넣는다. 


### 추가함수-바이트관련(SUBSTRB, LENGTHB)

#### SUBSTRB : 바이트크기로 계산하여 문자열을 잘라낸다.

SUBSTR은 문자의 개수를 세어 길이를 측정했다면 SUBSTRB는 바이트 크기를 계산하는 함수이다.

```sql
SELECT SUBSTRB('일이삼사오육칠팔구',4,3), SUBSTRB('abcdefg',1,3)
FROM dual;
```

> 오라클 DB에서 한글은 한글자를 표현하기 위해서 3바이트가 필요하다. 영어의 경우에는 한글자에 1바이트이기 때문에 차이점을 느끼기 어려울 수 있다.  
> 시작점설정도 바이트 단위로 측정되니 주의 해야한다. 한글로 작성된 문자열 '안녕' 을 자르고 싶을때 시작점을 2, 길이를 3으로 설정했다고 가정하면
> 한글의 문자하나의 바이트 크기는 3인데 시작점이 '안'의 중간지점에서 부터 잘라내기 때문에 결국엔 어떤 문자도 잘라내지 못하는 결과가 도출된다.  
> 이를 주의하여 잘 사용해야 한다. 

#### LENGTHB : 바이트크기의 단위로 문자열을 길이를 측정한다.

```sql
SELECT  LENGTHB('일이삼사오육'), LENGTHB('abcd')
FROM dual;
```

> `18`, `4`의 결과가 출력된다. ==> 한글 6글자 하나에 3바이트 , 영어 4글자 하나에 1바이트


## 변환함수

### TO_CHAR

문자로 바꿔주는 함수이다.

#### 날짜 -> 문자

날짜의 형식을 지정해줘야한다.


```sql 
SELECT TO_CHAR(SYSDATE, 'YYYY-MM-DD HH24:MI:SS DAY') AS 현재시간
FROM dual; 
```

> 사용자가 작성해준 형식으로 날짜를 문자로 변환해 준다.

> 날짜형 형식
>> - 년 : YYYY, YYY, YY, Y (EX. 2023년 ==> YYYY : 2023, YYY : 023, YY : 23, Y : 3)
>> - 월 : MM, MONTH, MON (EX. 12월 ==> MM:12, MONTH: DECEMBER, MON : DEC)
>> - 일 : DD, DDD, D (EX. 9일 == > DD : 09, DDD : <= 365, D : <= 7
>> - DDD는 1년중 며칠이 지났는가를 나타내는 수치, D는 해당 주의 며칠이 지났는가를 나타내는 수치임

추가적으로 년도를 설정할 때는 Y대신 R을 사용할 수 있는데 이 둘의 차이점은 Y는 현재 세기(20세기)를 기준으로 데이터를 변환해주고
R의 경우에는 이전 세기의 50년도 부터 현 세기의 49년도로 인식을 한다.

단, 년도가 정확히 명시되었을 때에는 명시한 그대로 나온다. 즉, 년도를 2개로 입력했을때 R과Y가 구분된다.

```
EX1)날짜데이터에 48-01-04 의 형식을 넣어주었다고 가정 (49년도 이하)
RRRR : 2048
YYYY : 2048
RR : 48
YY : 48

EX2)날짜데이터에 55-04-02 의 형식을 넣어주었다고 가정 (50년도 이상)
RRRR : 1955
YYYY : 2055
RR : 55
YY : 55

EX3)날짜데이터에 2355-04-02 의 형식을 넣어주었다고 가정 (년도에 4글자 입력)
RRRR : 2355
YYYY : 2355
RR : 55
YY : 55
```


#### 숫자 -> 문자

날짜와 다르게 형식을 지정하지 않아도 되지만, 형식을 지정해줘서 원하는 형식을 만들 수 있다.

> 숫자 형식 (예시 : 입력 12345678 가정)
>> - '999,999,999' : 12,345,678 / 뒤에서부터 형식에 맞게 작성
>> - '000,999,999' :  012,345,678 / 뒤에서부터 형식에 맞게 작성하며 형식에 입력되지 않는 수가 있으면 0으로 채워넣음
>> - '999,999,999.00' : 12,345,678.00 / 뒤에서 붙어 형식에 맞게 작성하며, 소수점자리수 지정
>> - '$9,999.999.999' : $123,456,789 / 돈을 표시하는 형식도 가능하다.

<strong style="color:red">!! 주의 </strong>

숫자형식을 작성할 때 0, 9를 구분해서 작성해야한다. 0은 해당위치에 숫자가 입력되지 않으면 0으로 채워넣는다.

입력되는 데이터보다 형식의 길이가 짧을 경우 구조를 이해하지 못하고 #### 과같은 이상한 값을 출력할 수 있다.



***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}