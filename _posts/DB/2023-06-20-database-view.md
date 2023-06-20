---
title:  "[DB] OracleDB 뷰"  
excerpt: "VIEW, 가상테이블"

categories: # 분류하고싶은 카테고리 입력
  - ORACLEDB
tags:     # 중요 키워드
  - [OracleDB]

toc: true
toc_sticky: true
published: true

date: 2023-06-20
last_modified_at: 2023-06-20
---


인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1]()
{: .notice--warning}

## VIEW

하나 이상의 테이블이나 다른 뷰의 결과 집합을 볼 수 있는 객체이다.

뷰를 사용하여 복잡한 쿼리문을 단순화 하고 필요한 정보만 취합하여 사용 가능해 쿼리문을 단순화 시킬 수 있고,
복잡한 쿼리를 뷰로 저장시켜 놓으면 검색을 통해 편리하게 사용 가능하며 ,뷰를 통해 다른 사용자가 데이터에 접근하기 용이하도록 만들 수 있다.

뷰는 가상 테이블이라고 말하는데 주의해야할 점은 데이터가 저장되는게 아닌 쿼리문이 저장되는 것이다.

### 뷰의 생성

> 생성 명령어

```sql 
CREATE [OR REPLACE] VIEW 뷰명
AS
SELECT 컬럼 FROM 테이블명 WHERE 조건;
```

"뷰명"은 생성할 뷰의 이름을 나타내며, AS 뒤에 나오는 쿼리문이 생성되는 뷰에 저장될 쿼리문입니다. 

쿼리문은 단순히 쿼리문을 나타내기 위해 작성된 것으로 형태는 달라질 수 있고, 필요에 맞게 쿼리문을 작성하면 됩니다.

"뷰명"은 생성시 뷰의 이름은 테이블인지 뷰인지 구분하기 위해 접두사,접미사로 뷰임을 알려주는 문구를 명시하는 것이 좋다.

반드시 작성해야 하는것은 아니며 상황에 따라 작성하면된다. (EX. vw_view명, view명_vw .......)

CREATE만 작성하면 생성 시에만 사용하며, 만약 동일한 이름의 뷰가 존재하면 생성되지 않는다. 

CREATE OR REPLACE로 작성하면 동일한 이름의 뷰가 존재하면 현재 작성한 뷰로 수정된다.

> 뷰 조회

```sql
SELECT * FROM 뷰명;
```

뷰에 저장된 쿼리문을 실행한 결과를 가져온다. ( 테이블 조회 )


### VIEW의 특징 

`VIEW는 생성 당시 작성해준 기준에서 데이터가 들어가는 것이 아니라 쿼리문 자체(TEXT)가 저장되는 형식이다`

❗ 주의

VIEW에서 사용된 TABLE은 서로 연결되어있다.

그러므로 VIEW를 통한 INSERT문, UPDATE문을 작성했을 경우 VIEW가 참조하는 TABLE에도 영향을 끼치게된다.

하지만, 여기서 주의해야할점은 VIEW로 모든 데이터를 입력, 수정가능하지 않다. 해당 테이블에서 수정,입력이 가능한 데이터는 VIEW에서 사용한 데이터로 한정 되어있다. 

삭제 또한 마찬가지이다. VIEW에서 조회되지 않은 데이터를 VIEW를 통해서 삭제할 수 는 없다. 만약 TABLE에 30개의 레코드가 삽입되어있고, 해당 TABLE을 참조하여 생성한
VIEW에는 0개의 레코드가 존재한다고 가정하면, VIEW를 통해서 삭제할 수 있는 데이터는 0개이다.

<p><details>
<summary style="color:red;">테스트를 위한 테이블 생성 코드</summary><!-- summary 아래 한칸 공백 필요 -->

<pre><code class="language-sql">

CREATE TABLE 주문 (
  주문번호 NUMBER,
  고객번호 NUMBER,
  주문일자 DATE,
  총금액 NUMBER
);


INSERT INTO 주문 (주문번호, 고객번호, 주문일자, 총금액) VALUES
(1, 101, TO_DATE('2023-05-01', 'YYYY-MM-DD'), 50000);

INSERT INTO 주문 (주문번호, 고객번호, 주문일자, 총금액) VALUES
(2, 102, TO_DATE('2023-06-10', 'YYYY-MM-DD'), 80000);

INSERT INTO 주문 (주문번호, 고객번호, 주문일자, 총금액) VALUES
(3, 103, TO_DATE('2023-07-15', 'YYYY-MM-DD'), 35000);

CREATE TABLE 고객 (
  고객번호 NUMBER,
  고객이름 VARCHAR(50),
  전화번호 VARCHAR(50),
  이메일 VARCHAR(50)
);

INSERT INTO 고객 (고객번호, 고객이름, 전화번호, 이메일) VALUES
(101, '홍길동', '010-1234-5678', 'hong@example.com');

INSERT INTO 고객 (고객번호, 고객이름, 전화번호, 이메일) VALUES
(102, '김철수', '010-9876-5432', 'kim@example.com');

INSERT INTO 고객 (고객번호, 고객이름, 전화번호, 이메일) VALUES
(103, '박영희', '010-5555-5555', 'park@example.com');

</code></pre>
<hr/>

</details></p>

> 주문내역을 확인하는 뷰

```sql 
CREATE VIEW 주문내역 AS
SELECT O.주문번호, C.고객이름, O.주문일자, O.총금액
FROM 주문 O
JOIN 고객 C ON O.고객번호 = C.고객번호;
```

고객별로 주문내역을 확인하는 쿼리문을 VIEW에 작성하고 

```sql 
SELECT * FROM 주문내역;
```

위의 쿼리문을 통해서 뷰에 저장된 주문내역을 간단하게 출력가능하다.

***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🧐

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}