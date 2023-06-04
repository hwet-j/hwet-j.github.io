---
title:  "[IDE/Web] 이클립스 JSP파일 에러 해결"  
excerpt: "javax.servlet 에러"

categories: # 분류하고싶은 카테고리 입력
  - IDE
tags:     # 중요 키워드
  - [IDE, Java, Web]

toc: true
toc_sticky: true

author: Hwet

date: 2023-05-27
last_modified_at: 2023-05-27
---

인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1](https://m.blog.naver.com/bgpoilkj/221656227898)
{: .notice--warning}

## JSP파일 에러 

The superclass 'javax.servlet.http.Httpservlet' was not found on the java buid path' 라는 에러가 발생한다.

슈퍼클래스인 javax.servlet.http.Httpservlet를 찾을 수 없다는 에러이다.

### 해결방법

> Build path -> Configure Build Path.. 클릭

![Build path](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/59aeca6a-79d6-492c-bdaa-65b53cc0212f)

> Java Build Path -> Libraries -> Add External JARs.....

![JARs설정](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/2db6015a-1ab3-49aa-97f3-a9000ed0d12e)

> 설치한 톰캣의 위치로 가서 lib 폴더의 servlet-api.jar, jsp-api.jar 파일을 추가해준다.

![완료](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/958418ad-c9f9-4f7d-99af-9706346ba644)

`해결`

***

<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}