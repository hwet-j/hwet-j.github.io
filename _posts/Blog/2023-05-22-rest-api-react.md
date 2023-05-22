---
title:  "[Github 블로그] REST API를 사용하여 이미지 가져오기" 

categories:
  - Git
tags:
  - [Git, RESTAPI]

toc: true
toc_sticky: true

author: Hwet

date: 2023-05-22
last_modified_at: 2023-05-22
---


[참고/출처](https://velog.io/@131ryuji/Github-API%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%98%EA%B8%B0)
[참고/출처](https://docs.github.com/ko/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)
{: .notice--warning}

## REST API를 사용하여 이미지 가져오기

### REST API를 사용해야 하는 이유

- 이미지 파일이 많아지면 로컬 저장소의 용량이 부족해질 수 있다.
- Github에서 100MB보다 큰 파일을 차단하고 Repository의 크기도 제한되어있기 때문에 이런 문제를 해결 할 수 있다.
- 이미지 파일을 백업하지 않으면 데이터 손실이 발생할 수 있다.
- 로컬 저장소에 저장하면 이미지 파일의 공유가 힘들어진다.

### Github API 사용하기

1. 깃허브 토큰얻기 (2021년도인가 부터 비밀번호가아닌 토큰을 사용)
  - 1. 로그인 후 우측 상단에 프로필 클릭 후 `Settings` 클릭 
  - 2. 왼쪽 사이브 바의 `Developer settings` 클릭
  - 3. 왼쪽 사이브 바의 `Personal access tokens` -> `Fine-grained token` 클릭
  - 4. 토큰 이름과 만기일 설정(옵션 Description)
  - 5. 조건에 따른 Repository 접근 설정

***
<br>

    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}