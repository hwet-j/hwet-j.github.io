---
title:  "[IDE] Eclipse - Source not found"
excerpt: "Java 클래스파일 확인 오류"

categories:
  - IDE
tags:
  - [Java, Eclipse]

toc: true
toc_sticky: true

date: 2023-05-20
last_modified_at: 2023-05-20
---

인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1](https://treasurebear.tistory.com/42)
{: .notice--warning}


## JDK가 제공해주는 Source파일 확인

개발을 하다보면 JDK가 제공해주는 클래스 파일을 보고싶은 경우가 있는데 이 때 Class File Editor - Source not found 와 같이 뜨며 파일을 볼 수 없는 경우가 있습니다.

### 해결방법 

> 자바 클래스 소스파일의 경로가 없음

  ![파일을 찾을수없음](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/c9d5e156-6703-4220-bf58-f029039fd7ff)

> Attach Source 클릭

  ![Attach Source](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/2dbc5231-efdf-4cf2-91ca-4fc2b40a2afd)

> External File 클릭 -> 경로지정 (각자 설치한 경로의 src.zip파일)

  ![External File](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/c4406cd5-11ea-46a6-9ae9-ff0cb64cfb67)





***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}