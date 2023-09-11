---
title:  "[Web] Spring에서 Kakao API를 사용하여 로그인 구현하기"  
excerpt: "API를 활용한 로그인 기능"

categories: # 분류하고싶은 카테고리 입력
  - Spring
tags:     # 중요 키워드
  - [KakaoAPI]

toc: true
toc_sticky: true
published: true

author: Hwet

date: 2023-08-21
last_modified_at: 2023-08-21
---

인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.  
[참고](https://bcp0109.tistory.com/379)
[참고](https://velog.io/@hwan2da/Spring-%EC%B9%B4%EC%B9%B4%EC%98%A4-%EB%A1%9C%EA%B7%B8%EC%9D%B8-%EA%B5%AC%ED%98%84%ED%95%98%EA%B8%B0)
{: .notice--warning}


# 수정 진행 중입니다.


## Kakao API를 사용하여 로그인/회원가입 하기

작동원리나 이론적인 내용은 완벽하게 이해하지 못한 상태에서 구현에만 초점을 두고 작성하고 진행하였습니다.

이후에 작동원리나 이론을 이해하면 따로 정리하도록 하겠습니다. 


### 개발환경

Spring boot 2.7.14
Gradle
JAVA 17
Thymeleaf
Lombok
Spring Web
Spring Data JPA

### Kakao API 등록하기

> 카카오 개발자 홈페이지 접속

[카카오 개발자 페이지](https://developers.kakao.com/)

> "내 어플리케이션" 이동

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/1128f923-29f5-46f9-9190-679ab16c324f)

> "어플리케이션 추가하기"

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/ea90327b-cb51-4621-a219-b4ec7505c91c)

사업자가 아니더라도 적절한 이름 설정하여 생성가능

> "카카오 로그인"

자신의 도메인을 등록해줍니다. 


















*** 



***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}