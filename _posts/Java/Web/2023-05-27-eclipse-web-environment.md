---
title:  "[IDE/Web] 이클립스 Java Web 환경 구축"  
excerpt: "초기 구축방법"

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
[참고1](https://mjn5027.tistory.com/56#%ED%86%B0%EC%BA%A3%20%EC%84%A4%EC%B9%98%20%EB%B0%A9%EB%B2%95)
{: .notice--warning}

## Eclipse에서 Web환경 구축하기

기본적인 자바의 실행은 가능한 상태에서 부터 작성하는 글입니다. (즉, JDK의 설치 및 연동이 완료된 상태)

### 아파치 톰캣 (Apache Tomcat) 설치

[아파치톰캣 홈페이지](https://tomcat.apache.org/download-90.cgi)

아파치톰켓 설치는 위의 홈페이지에서 다운 받으면 된다. 나는 Tomcat9 를 설치 했다. 

![아파치설치](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/96923d86-4209-4085-be77-01f5ceef5e57)

- 다운로드 탭에서 원하는 버전을 클릭한다.
- Binary Distributions -> Core -> 32-bit/64-bit Windows Service Installer 클릭
- 다운이 완료된 Installer를 실행하여 설치한다.
- 포트는 원하는 포트를 입력하고 넘어간다. (정확히 모르니 검색해보길 추천)
- 설치중, JDK가 컴퓨터에 잘 설치되어 있다면 알아서 경로를 찾아준다. 그렇지 않다면 JDK부터 다시 설정을 완료할 것을 추천한다.

#### 아파치 톰켓이란?

아파치 톰캣은 톰캣이 아파치의 기능 일부를 가져와서 제공해주는 형태로 WAS(Web Application Server)라고 부른다.

간단하게 아파치는 정적인 웹페이지를 처리하고, 톰캣은 동적인 웹페이지를 처리하는데 아파치 톰캣을 같이 쓰는 거라고 보면된다.

우선, 지금은 웹페이지를 구현할수있게 해주는 웹 어플리케이션 서버이라고 생각하고 넘어가겠다. 

*** 

### 이클립스와 아파치톰캣 연동

- `Java EE(모드)에서 Create a Dynamic Web project 클릭 또는 File -> New -> Dynamic Web project 클릭` 

![img](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/51c7d2b1-9f4f-455e-976d-45e2be36abad)

- `Target runtime -> New Runtime.. -> 설치한 Tomcat 버전으로 설정 -> Next -> 톰캣 경로와 JRE 설정`

톰캣의 경로는 자신이 설치한 톰캣 경로 (C:\Program Files\Apache Software Foundation\Tomcat 9.0)로 설정해준다.

![RuntimeSet](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/1f32b0c3-8ed4-4772-9bf8-e519a09a970e)

- `프로젝트 생성을 완료하고, WebContent에 .jsp파일 또는 .html 파일을 생성해준다.(내 파일이 작동되는것인지 글 문구 하나 작성하여 테스트)`

![파일생성](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/5f8d1e94-0b3f-4fc4-83f8-236448692881)

- `프로젝트 폴더에서 우클릭 -> Run As -> Run on Server`

![서버실행](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/e54ad47f-45bd-45de-bd75-7d80cff2b56f)

- `이러면 작동이 될것이다. 만약 작동되지 않고 Could not launch external web browser.... 에러가 발생할 수 있다.`

웹 브라우저 경로 문제로 windows -> preferences -> General -> Web Browser 로 들어간다.

자신이 설치한 브라우저 중에 사용하고 싶은 브라우저를 선택하고 그에 맞는 경로를 지정해준다. 


***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}