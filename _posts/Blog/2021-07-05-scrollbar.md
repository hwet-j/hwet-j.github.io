---
title:  "[Github 블로그] 스크롤바 꾸미기" 

categories:
  - Blog
tags:
  - [Blog, jekyll, Liquid, HTML, minimal-mistake]

toc: true
toc_sticky: true
 
date: 2021-07-05
last_modified_at: 2021-07-05
---

## 🚀 스크롤바 꾸미기

블로그 방문자님께서 스크롤바 커스텀하는 방법에 대해 물어봐주셔서 블로그에 업로드 해본다. 나도 예전에 다른 블로그에서 보고 따라했던건데 출처를 다시 찾아보려니 못 찾겠다..😂

### 📜default.html

📂_layout 폴더에 있는 📜default.html 를 열어봅시다! (이것은 minimal-mistake 기준이며 꼭 📜default.html 아니더라도 <u>html 의 \<head> 태그가 있는 곳!!!</u>) 그리고 아래 코드를 `<head>` 태그 내에 넣어주면 된다.

```html
    <style> 
      ::-webkit-scrollbar{width: 16px;}
      ::-webkit-scrollbar-track {background-color:#4b4f52; border-radius: 16px;}
      ::-webkit-scrollbar-thumb {background-color:#5e6265; border-radius: 16px;}
      ::-webkit-scrollbar-thumb:hover {background: #ffd24c;}
      ::-webkit-scrollbar-button:start:decrement,::-webkit-scrollbar-button:end:increment 
      {
          width:12px;height:12px;background:transparent;}
      } 
    </style>
```

나의 스크롤바 코드는 위와 같다. 이 코드는 📜default.html 의 `<head>` 태그 내에 있다. 근데 이 코드는 왜인지 모르겠지만 **컴파일 에러, 즉 빨간 줄이 뜬다.** 빠른 수정을 할 수 없다며 중괄호 부분에 에러가 뜨는데 왜 뜨는지 잘 모르겠지만.. 스크롤바가 블로그 서버에 반영되는데 전혀 문제가 없다. 이 상태에서 📜default.html 의 git commit 도 잘 된다! 에러가 떠도 걱정 하지 않으셔도 괜찮을 것 같아요! 

- track, thumb, 등등 다 스크롤의 구성 요소들이다. 
- `border-radius` : 모서리 둥글게 할 정도. 
- `background-color` : 스크롤 색상 지정.. 저렇게 하드 코딩하면 안되는데 ㅠㅠ.. 색상을 변수 파일에 지정해두고 사용하실 것을 추천드립니다!

***
<br>

    🌜 개인 공부 기록용 블로그입니다. 오류나 틀린 부분이 있을 경우 
    언제든지 댓글 혹은 메일로 지적해주시면 감사하겠습니다! 😄

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}