---
title:  "[Github] Merge, Rebase 알아보기"
excerpt: "Merge와 Rebase의 기능과 둘의 차이"

categories:
  - Git
tags:
  - [Github]

toc: true
toc_sticky: true

author: Hwet

date: 2023-09-13
last_modified_at: 2023-09-13
---


[참고/출처](https://firework-ham.tistory.com/12)

[참고/출처](https://velog.io/@sweet_sumin/Git%EC%9D%98-Merge%EC%99%80-Rebase%EC%9D%98-%EC%B0%A8%EC%9D%B4)
{: .notice--warning}

## 📌 깃허브 Merge, Rebase

### 🖇️ Merge 

`merge`는 두 개의 브랜치를 합친다는 의미이다. 이때 Github의 브랜치와 브랜치가 합쳐질 수 있고, Github Branch와 로컬 컴퓨터에 저장된 정보일 수 있다.

merge는 크게 두 가지 방식이 존재한다

> Fast-Forward 방식과 Non Fast-Forward 방식

#### ✏️ Fast-Forward

협업 또는 브랜치를 분리하는 과정에서 `main` 브랜치에서 `A-branch`를 생성했다고 가정하면 초기 Commit 상태는 동일한 상태일 것이다.  
여기서 `A-branch`에서 특정한 기능을 작업하고 Commit을 진행하였고, `main`에서는 어떠한 작업도 하지 않았다고 생각하면 `A-branch`의 Commit 내역이 더욱 많고 앞서있다고 할 수 있다.

이때, 더 앞서있는 브랜치 위에서 `A-branch`를 기준으로 병합하는 것을 <u>Fast-forward</u>라고 한다.  
(이 과정에서는 충돌이 일어나지 않는다.)

<div class="mermaid"> 
gitGraph
  commit id: "A"
  commit id: "B"
  commit id: "C"
  branch A-branch
  commit id: "D"
  commit id: "E"
  checkout main
  merge A-branch id: "Merge" tag: "A-B-C-D-E-F"
</div>

이때 `main`의 Commit 내역과 `A-branch`의 Commit 내역은 `A-B-C-D-E-F`로 동일하다.(Merge Commit을 F로 봤을때)

일반적으로 `A-branch`는 완벽히 동일한 정보를 가지고 있기 때문에 병합 이후에 삭제하는것이 좋다.


#### ✏️ Non Fast-Forward

협업 또는 브랜치를 분리하는 과정에서 `main` 브랜치에서 `A-branch`를 생성했다고 가정하면 초기 Commit 상태는 동일한 상태일 것이다.  
여기서 `A-branch`에서 특정한 기능을 작업하고 Commit을 진행하였고, 
`main`에서도 특정한 작업을 진행한 상태라면 누가 더욱 앞서있다고 알 수 없는 상태이다.

이런 상황에서 병합하는 것을 <u>Non Fast-forward</u>라고 한다.  

Non Fast-forward 에서는 충돌이 일어날 가능성이 있는데, 팀원A와 팀원B가 완벽하게 다른 기능을 담당하여 동일한 파일을 만지지 않는다면 발생하지 않겠지만,
만약 동일한 파일, 동일한 부분의 설정이 변경되었을 경우에 충돌이 일어나게 된다.

이런 상황에 오게되면 해당 충돌이 일어난 부분을 작업한 개발자들이 모여 어떻게 수정을 진행하여 병합할 것인가 설정해 줘야한다.


<div class="mermaid"> 
gitGraph
       commit id: "A"
       commit id: "B"
       branch A-branch
       checkout A-branch
       commit id: "C"
       checkout main
       commit id: "D"
       checkout A-branch
       commit id: "E"
       checkout main
       commit id: "F"
       checkout main
       merge A-branch id: "Merge" tag: "A-B-D-F-G"
</div>

Non Fast-forward 방식은 Fast-forward와 다르게 Commit 내역이 동일하지 않다. 

`main`의 Commit 내역은 `A-B-D-F-G` 이며, `A-branch`는 `A-B-C-E-G`가 된다.

Non Fast-forward로 병합한 이후에도 최종 결과는 동일하므로 `A-branch`는 삭제해주는 것이 좋지만,
주의하여 삭제 해야한다. `A-branch`에만 있는 Commit 내역도 함께 삭제되므로 해당 Commit 기록이 필요없다고 판단되었을 때 제거 해야한다. 

### 🖇️ Rebase











***
<br>

    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}