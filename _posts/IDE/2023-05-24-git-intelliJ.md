---
title:  "[IDE/Gitgub] Github로 협업하기 (Fork)"
excerpt: "Github/IntelliJ를 사용해서 협업해보기"

categories:
  - IDE
tags:
  - [Github, IntelliJ]

toc: true
toc_sticky: true

date: 2023-08-24
last_modified_at: 2023-09-12
---

인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1](https://rebornbb.tistory.com/entry/GIT-intelliJ%EC%97%90%EC%84%9C-GitHub%EC%97%90-%EA%B8%B0%EC%A1%B4-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-%EC%97%85%EB%A1%9C%EB%93%9C%ED%95%98%EA%B8%B0)
{: .notice--warning}


## Github로 협업하기

Github로 협업하는 방식에는 여러가지 있지만, Fork를 통해서 협업하는 방법에 대해서 작성할 것이며,
설명은 팀원, 팀장(협업 레포지토리)로 진행할 예정이며, Fork된 정보를 IntelliJ를 통해서 Github연동 하는 방법을 작성하고자 한다.


### https://github.com/ 로그인 

자신(팀원)의 아이디로 로그인한다.

### 협업할 레포지토리(팀장)에 접근하여 Fork 하기

예시는 저의 레포지토리 주소로 들겠습니다. https://github.com/hwet-j/pearl 에 접속하여 우측 상단에 Fork 버튼한다.

![Fork](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/866ddd2f-1abe-4275-9d91-0406b57278b5){: width="40%" height="40%"}

Fork를 클릭하게 되면 아래와 같은 화면이 뜨는데 Repository name을 설정해주고 생성주면된다.

![포크화면](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/fa3edefa-bb01-4a86-b767-201b7a1c3c2c){: width="40%" height="40%"}

Repository name은 자기 자신의 저장소에 저장할 이름을 설정하는 정보이다.

설정 후 `Create fork` 클릭하고 자신의 Github Repository에 설정한 정보로 생성되었는지 확인한다.

생성되었다면 Github에서 협업할 준비가 완료된 것이다.


### IntelliJ에서 Github 프로젝트 가져오기

1. IntelliJ 실행

2. 프로젝트 실행화면으로 이동 (프로젝트가 실행된 상태면  "파일 - 프로젝트 닫기")

3. VCS에서 받기

4. GitHub 클릭

5. 로그인

6. 로그인이 완료되면 자신의 Github에 있는 Repository목록이 나오는데 여기서 Fork하면서 생성한 Repository선택 후 복제


### IntellIJ에서 연동된 Github Repository에 Commit & Push 하기

1. 업로드를 하기 위해서는 수정된 정보가 있어야함

2. Commit하기 -> "Ctrl + K", "Alt + 0" 입력 또는 IntelliJ의 "Git:Branch - Commit" 클릭

3. 수정된 정보에서 업로드할 정보를 선택

4. Commit 또는 Commit & Push 클릭

5. Commit을 선택한 경우 -> Push 별도 진행 필요

6. Push하기 -> "Ctrl + Shift + K" 입력 또는 IntelliJ의 "Git:Branch - Push" 클릭

7. Commit내용을 확인할 수 있으며 어떤 데이터가 올라가는지 확인가능. 올리려면 Push 클릭

### IntellIJ에서 연동된 Github Repository의 정보 가져오기 (로컬 프로젝트 최신화)

이클립스 및 Git 명령어의 pull 기능을 해야하는데 IntelliJ는 프로젝트 업데이트로 가능

1. "Ctrl + T" 입력 또는 IntelliJ의 "Git:Branch - Project Update" 클릭

2. Merge와 Rebase를 선택해야 한다. 

3. Merge 선택을 추천한다. (Github에 있는 브랜치와 로컬 브랜치를 합치겠다는 의미)

`Merge와 Rebase에 대한 정보는 별도의 포스트 참고`





> 관련 용어

- Branch : 하나의 Repository내에서 별도의 독립적인 공간을 생성하기 위해 Branch를 생성한다.
  - 기능별로 구현할 때 Branch를 생성하여 작업이 완료된 이후 master에 병합하고 삭제하는 방식으로 사용가능
  - 하나의 Repository를 공유하여 협업진행 시에 개인별로 별도의 Branch를 생성하여 작업이 가능

- Commit : 세이브 포인트, 저장을 원하는 파일들을 묶어서 커밋
  - 구현한 기능 별로 기록을 하며 Commit을 진행하면 좋다.

- Push : 로컬 저장소 내에서 작업한 Commit내역을 Github(원격 저장소)에 옮김
  - 마지막으로 Push한 이후에 Commit한 내역을 저장소에 업데이트 시켜준다.
  - 일반적으로 개인의 Branch하나에서 작업하기 때문에 문제가 발생할 일이없지만 발생한다면 Pull을 받지않아 발생한다.

- Pull requests : 팀장(Fork한 Repository)에게 자신의 정보(Commit 내역)를 pull하라고 요청하는 기능 
  - 자기 자신의 Branch끼리도 가능하며 A Branch의 정보를 B Branch로 Commit내역을 보내는 작업이라고 보면된다.

아직 Github에 공부하며 작성중이므로 이해가 안되거나 안되는 부분이 있으시면 다른 정보를 찾아보시는것을 추천드립니다.











***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}