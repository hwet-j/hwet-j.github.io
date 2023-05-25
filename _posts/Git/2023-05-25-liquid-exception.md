---
title:  "[Git/Blog] Liquid Exception 에러" 

categories:
  - Git
tags:
  - [Git, RESTAPI]

toc: true
toc_sticky: true

author: Hwet

date: 2023-05-25
last_modified_at: 2023-05-25
---


[참고/출처](https://iamheesoo.github.io/blog/gitblog-sol-jekyll02)
[참고/출처](https://github.com/jekyll/jekyll/issues/5458)
{: .notice--warning}

## Liquid Exception: Liquid syntax error

블로그를 작성하다가 깃허브에 올려도 웹페이지 내에서 업데이트가 되지않아 확인했더니, Build 에러가 발생하고 있었다.

![LiquidException](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/82661b9a-b552-4021-b71e-0ba72583f95e)

### 에러 원인

Jekyll에서 사용되는 liquid는 `\{\{`와 `\}\}`를 escape 문자로 사용하기 때문에 발생하는 문제이다. 

### 해결 방법

생각보다 엄청 간단하다. 

`\{\{`를 `{ {`로 바꿔주면 된다. 즉, 중괄호와 중괄호 사이에 공백(space)를 넣어주면 된다.

다른 방법으로는 {% raw %}와 {% endraw %}로 감싸주면 된다.

***
<br>

    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}