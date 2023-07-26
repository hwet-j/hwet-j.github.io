---
title:  "[Web/JSP] DB연동 과정중 ClassNotFoundException 문제"  
excerpt: "DB 연동중에 발생하는 ClassNotFoundException"

categories: # 분류하고싶은 카테고리 입력
  - Error
tags:     # 중요 키워드
  - [HTML, JSP]

toc: true
toc_sticky: true
published: true

author: Hwet

date: 2023-06-28
last_modified_at: 2023-06-28
---

인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1]()
{: .notice--warning}


## JSP와 DB간에 데이터 주고받기?

환경은 이클립스의 Dynamic Web Project 내에서 JSP와 DATABASE를 연동하는 과정에서 발생했던 ClassNotFoundException을 해결하는 방법을 작성한다.

이 문제는 기본적으로 다른 상황에서 문제가 없었을 때를 기준(Java Project에서 DB연동 등...)으로 작성되었습니다. 

ClassNotFoundException 문제는 특정 파일을 정확히 지칭하지 않고 호출한 Class가 없었을 경우 발생하는 문제로 다른 상황에서도 동일한 문구를 띄울 수 있습니다.

일반적인 기본개념은 동일합니다. 지금 불러오는 코드에서 필요한 클래스가 있는 파일을 프로젝트에 설정해주면 됩니다.

### 해결방법

DB 연동중 발생하는 ClassNotFoundException는 DB와 연관되어있는 jar파일을 읽어올 수 없어서 발생하는 문제이므로 이 파일을 직접적으로 넣어주거나 build 설정을 해주면 된다.

저는 오라클 11g 버전을 사용하고 있어 ojdbc5 또는 ojdbc6을 사용하면 됩니다. [오라클홈페이지](https://repo1.maven.org/maven2/com/oracle/database/jdbc/)에서 각자에 맞는 버전을 찾아 사용하시면 될거같습니다.

다운 받은 jar파일을 직접적으로 `프로젝트 - WEB-INF - lib` 안에 넣어주면 해결됩니다.

또는 `프로젝트 - 우클릭 - Build Path - Configure Build Path...` 클릭 `Java Build Path - Library - Add External JARs..` 를 클릭하여 자신이 다운 받은 jar파일을 넣어주고 저장한다.




*** 



***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}