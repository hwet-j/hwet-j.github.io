---
title:  "[Spring/lombok] ToString 사용시 발생하는 StackOverFlow"  
excerpt: "ToString, Data 어노테이션을 사용하면서 발생되는 무한재귀 현상"

categories: # 분류하고싶은 카테고리 입력
  - Error
tags:     # 중요 키워드
  - [Lombok, Spring]

toc: true
toc_sticky: true
published: true

author: Hwet

date: 2023-09-12
last_modified_at: 2023-09-12
---

인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1]()
{: .notice--warning}


##  ❓ 발생 시점 및 원인

Lombok 사용중, Entity간에 서로 참조하는 두 객체에 @ToString 또는 @Data 어노테이션이 작성되어있는 상태에서 발생되었다.

데이터가 잘 들어갔는지 확인하기 위해 @ToString 어노테이션을 사용해 확인하려고 하면서 알게되고 발생한 에러이다.

이 문제는 엔티티를 단순 조회한다고 해서 발생하는 문제는 아니다. A,B 엔티티가 서로를 참조 하고 있을 경우에 다른 Entity를 불러오면서(Json으로 변환) 발생하는 문제이다.

엔티티를 Json으로 변환하는 과정에서 A의 ToString이 B를 불러오고 B의 ToString이 A를 불러오는 작업을 반복하면서 
양방향 매핑으로 인한 무한 재귀현상이 발생하게되는것이다.


```java
java.lang.StackOverflowError Create breakpoint
at com.pits.auction.tostring. MusicAuction.toString(MusicAuction.java:7) at java.base/java.lang.String.valueOf(String.java:2951) at com.pits.auction.tostring. Member.toString(Member.java:8) at java.base/java.lang.String.valueOf(String.java:2951) at com.pits.auction.tostring. MusicAuction.toString(MusicAuction.java:7) at java.base/java.lang.String.valueOf(String.java:2951) at com.pits.auction.tostring. Member.toString(Member.java:8 at
java.base/java.lang.String.valueOf(String.java:2951)
at com.pits.auction.tostring. MusicAuction.toString(MusicAuction.java:7)
```


### ❗해결방법

> JsonIgnore

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/a3f76993-5026-42fd-958e-e1c57b10964a){width=35% height=35%}

가장 간단하게 해결할 수 있는 방법은 참조하는 연관 객체들에 @JsonIgnore 어노테이션을 작성하여 해결 가능하다.

말그대로 Json을 직렬화 할때 해당 객체는 무시하고 진행하겠다는 것이다. Json 직렬화 때 사용되지 않으니 순환참조가 발생하지 않는다.

하지만, 해당 객체를 Json화하여 사용한다거나 사용해야할 다른 상황이 있을 때 사용하지 못할 가능성이 매우 높다.

실제로 Auction 프로젝트를 진행하면서 글목록을 무한 스크롤을 통해 구현하였는데, 이때 추가되는 목록들을 Json 형식으로 받아와서 진행했는데

특정 필드가 들어오지 않는 (JsonIgnore을 적용한 필드) 문제가 발생하여 다른 방식으로 순환참조를 해결하였다.

이런 문제들 때문에 순환참조를 방지하기 위해서 @JsonIgnore를 사용하는 것은 추천하지 않고 지양해야 한다고 한다.

> ToString 조건설정

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/be2bd2ce-60b9-4502-863b-908e4ba3f515){width=35% height=35%}

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/055b8e1f-51a3-4f68-95db-9238ffebef24)

@ToString 어노테이션에 조건을 설정하여 exclude 속성으로 양방향 매핑에 문제가 되는 필드를 ToString메서드에서 제외하도록 할 수 있다.

@ToString(exclude = "musicAuctions") 와 같이 설정해주면 해당 필드는 ToString 생성시에만 제외되며, Json타입으로 형변환할 당시에는 문제되지않는다.

> DTO 

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/6d985ea1-d99a-4017-be68-ab1ab3333a7d){width=35% height=35%}

데이터베이스의 정보를 사용시에 DTO와 같은 객체를 생성하여 참조 객체를 직접 담지 않고 필요한 정보만 사용하도록 설정하여 순환 참조가 발생하지않도록 방지 가능하다.

애초에 일반적으로 엔티티를 직접 사용해서 정보를 주고받게되면 테이블의 전체 정보를 담기 때문에 클라이언트에게 불필요한 정보를 전해줄 가능성이 있고,
수정을 원하지 않는 정보가 수정될 수도 있다. 

그렇기에 DTO를 활용하면, 양방향 매핑도 방지가능하며 필요한 컬럼만을 사용하기 때문에 더욱 효율적이고 클라이언트와 원하는 정보만 주고받을 수 있다.











*** 



***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}