---
title:  "[Web/lombok] PropertyNotFoundException"  
excerpt: "EL 사용시 해당 속성을 찾을 수 없을 때"

categories: # 분류하고싶은 카테고리 입력
  - Error
tags:     # 중요 키워드
  - [HTML, Lombok, Spring]

toc: true
toc_sticky: true
published: true

author: Hwet

date: 2023-08-11
last_modified_at: 2023-08-11
---

인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1]()
{: .notice--warning}


##  ❓ 발생 시점 및 원인

- 프로퍼티 이름이 잘못된 경우
- 접근 권한이 없는 경우
- 컬렉션에서 잘못된 키 또는 인덱스값을 사용하는 경우
- 객체가 null값인 경우

등이 있으며 말 그대로 속성값을 불러와 출력하려 했으나 존재하지 않는 데이터이기 때문에 발생하는 오류이다.


### ❗해결방법

발생 시점 및 원인에 해당하는 문제를 고려해 코드를 다시 분석해봐야한다. 

일반적으로 프로퍼티 이름을 옮겨적는 과정에서 에러가 발생했을 가능성이 크며, 여기서는 Lombok을 사용시 발생했던 문제를 작성하고자 한다.

`Lombok 이란? `

Java 언어를 위한 코드 생성 라이브러리로, 반복작업을 줄이고 코드의 가독성을 향상시키기 위해 사용되는 도구입니다.

어노테이션을 활용하여 컴파일 시점에서 자동으로 코드를 생성하거나 변경하도록 하여 개발자가 작성해야하는 반복작업을 줄여주는데 도음을 줍니다.

일반적으로 Getter, Setter, 생성자, toString() 등을 자동생성하는데 사용한다. 

> Lombok을 처음 사용하면서 PropertyNotFoundException가 발생한 문제

Spring구조를 이해하면서 Lombok 라이브러리를 처음 사용할 때 한번 선언하면 모든 데이터에 적용이 되는줄 알고 문제가 발생했다. 

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/2b42e892-2e79-4bdc-b56d-7cbad959a9bb)

위의 이미지와 같이 id값 위에 @Getter, @Setter를 작성해두었기 때문에 name필드에도 자동적으로 적용되는줄 알았다. 

이 상태에서 name 속성을 사용하려고 했더니 해당 속성을 찾을 수 없다는 에러가 발생해 여기서 이것이 문제인지 알지 못하고 다른 코드에서 에러를 찾았다.

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/5666d0c1-580e-4f51-af95-1777994ebcbb)

name필드에 적용하고 나니 해당 에러가 해결이 되었으며, 적용하고자 하는 모든 필드에 어노테이션을 작성하여 적용해야 하는것을 주의하자.







*** 



***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}