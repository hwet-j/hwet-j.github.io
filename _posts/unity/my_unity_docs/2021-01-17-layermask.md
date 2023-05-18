---
title:  "Unity C# > UnityEngine : LayerMask" 

categories:
  -  UnityDocs
tags:
  - [Game Engine, Unity]

toc: true
toc_sticky: true

date: 2021-01-17
last_modified_at: 2021-01-17
---

공부하면서 알게된 것만 정리합니다.😀
{: .notice--warning}


# 👩‍🦰 LayerMask

- LayerMask 는 32 비트의 `int`형이다.
  - 비트 플래그로, 비트로 각각의 아이템을 상징하며 구분된다.
    - 예
      - 몬스터 레이어는 000000000000000000000000<u>1</u>000000 👉 8 번 레이어 (1 << 8)
      - NPC 레이어는  0000000000000000000000000000<u>1</u>00 👉 3 번 레이어 (1 << 3)
  - 따라서 LayerMask는 위와 같이 32 개까지만 만들 수 있다.
  - `LeyerMask` 내에 연산자 오버로딩과 `int`로의 변환이 다 구현이 되어 있기 때문에 `LayerMask`라는 이름으로 편하게 접근할 수 있는 것 뿐이다.
- `public LayerMask a;` 해주면 유니티에서 Layer 를 선택할 수 있는 드롭다운 슬롯이 열린다.

```c#
            int mask = (1 << 8) | (1 << 9); // 8 번 레이어 + 9 번 레이어

            RaycastHit hit;
            if (Physics.Raycast(ray, out hit, 100.0f, mask))
            {
                Debug.Log(hit.collider.gameObject.name);
            }
```

- (1 << 8) **|** (1 << 9) 👉 000000000000000000000000<u>11</u>0000000
  - 즉 8번 레이어 혹은(`or`) 9 번 레이어(사진에선 "Monster"와 "Wall")를 가진 오브젝트들에 대해서만 Raycasting이 진행된다.
- 이처럼 비트 플래그를 활용한 비트 연산으로 Layer를 지정할 수 있다.
- (1 << 8) **|** (1 << 9) 은 십진수로 변환하면 768 이므로 `int mask = 768;` 도 가능하다.

## 🚀 함수

### ✈ LayerMask.GetMask

> public static int GetMask(params string[] layerNames);

```c#
            LayerMask mask = LayerMask.GetMask("Monster") | LayerMask.GetMask("Wall")/
            // int mask = (1 << 8) | (1 << 9); 와 동일

            RaycastHit hit;
            if (Physics.Raycast(ray, out hit, 100.0f, mask))
            {
                Debug.Log(hit1.collider.gameObject.name);
            }
```

- *LayerMask.GetMask(string)*
  - 해당 문자열로 저장된 LayerMask 즉, `int` 크기의 비트를 가져오게 된다.
  - 이처럼 함수를 사용해 편하게 레이어 이름으로 지정한 문자열로 비트를 가져올 수 있다.

***
<br>

    🌜 개인 공부 기록용 블로그입니다. 오류나 틀린 부분이 있을 경우 
    언제든지 댓글 혹은 메일로 지적해주시면 감사하겠습니다! 😄

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}