---
title:  "(C++) 투포인터 알고리즘, 슬라이딩 윈도우 알고리즘, Prefix Sum 구간합" 

categories:
  - Algorithm
tags:
  - [Algorithm, Coding Test, Cpp, Graph]

toc: true
toc_sticky: true

date: 2021-03-14
last_modified_at: 2021-03-14
---

- 투포인터 알고리즘 👉 구간이 가변적 길이
- 슬라이딩 윈도우 👉 구간이 고정적 길이

## 🚀 투포인터 알고리즘

<iframe width="1912" height="800" src="https://www.youtube.com/embed/iSjvuixMPmQ" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

> 어떤 특정 조건을 만족하는 **🔥연속 구간🔥**을 구할 때 `O(N)` 으로 풀 수 있도록 도와주는 알고리즘

2 개의 포인터를 사용하여 구간의 길이를 ***가변적*** 적으로잡아가며 특정 조건을 만족하는 구간을 찾는다. 모든 연속 구간을 잡는다면 O(N^2)이 될 것이지만 투 포인터 알고리즘을 사용하면 O(N) 의 시간복잡도로 풀 수 있다. 두 포인터 각각 `N`번 움직이기 때문에 `N` + `N` = `2N` 번의 연산이 있는 셈이다. 즉, O(2N) = O(N) 이 된다. 어떤 구간을 찾는 문제일 때 입력 크기가 아주 크다면 **투 포인터 알고리즘** 을 고려하여 풀자! 

2 개의 포인터 中 `right` 포인터는 어떤 조건일 때 증감시키고, `left` 포인터는 어떤 조건일 때 증감시킬지 결정해야 한다.
{: .notice--warning}

- 투포인터 알고리즘 문제의 유형
  1. 포인터 2개가 같은 방향으로 진행하는것 
  2. 포인터 2개가 양끝에서 시작하여 반대로 진행하는 것
  3. 포인터 하나는 한쪽 방향으로만 진행하고, 다른 포인터는 양쪽으로 이동하는 것

<br>

### 🔥 포인터 2개가 같은 방향으로 진행하는 경우

#### ✈ 코드1 (백준 2003번 문제 : 수들의 합 2)

> 두 포인터 모두 "오른쪽"으로 이동함

```cpp
#include <iostream>
#include <vector>
using namespace std;

int answer = 0;
int main() {
    int N, M;

    cin >> N >> M;
    vector<int> arr(N);
    for (int i = 0; i < N; ++i)
        cin >> arr[i];

    int start = 0;
    int end = 0;
    int sum = arr[0];
    while (end < N) {       
        if (sum < M) { // sum 이 더 커져야 하는 상황. 즉 end 포인터가 더 오른쪽으로 가야 한다.
            end++;
            if (end < N)
                sum += arr[end];
        }
        else if (sum > M) { // sum 이 더 작아져야 하는 상황. 즉 start 포인터가 더 오른쪽으로 가야 한다.
            sum -= arr[start];
            start++;
        }
        else if (sum == M) { // sum 이 M 과 같아졌으므로 현재의 star, end 로는 이 다음부터는 [start, end + 1] 도, [start + 1, end] 도 답이 될 수 없다. 두 포인터 다 더 오른쪽으로 한 칸씩 이동시킨다. 
            sum -= arr[start];
            start++;   
            end++;
            if (end < N)
                sum += arr[end];
            answer++;
        }
    }

    cout << answer;
}
```

> [[백준 2003][🤍3] 수들의 합 2 (투포인터 알고리즘)](https://ansohxxn.github.io/boj/2003/)

- **구간의 모든 원소의 합이 M 이 되는 구간을 구해야 한다.**
- **두 포인터가 같은 방향으로 움직이므로 첫 번째 원소 인덱스부터 시작**
{: .notice--warning}

- 현재 구간의 합이 `M`보다 작으면
  - 아직 `M`보다 작으니까 `M`에 근접하도록 합을 증가시키기위해 구간에 다른 수를 더 포함해야 한다. 
    - ex) 1 [2 3 4] 5 6 👉 1 [2 3 4 5] 6
    - `start`는 냅둔다.
    - `end` 를 증가시킨 후 sum 에 arr[end] 를 더한다.
- 현재 구간의 합이 `M`보다 크면 
  - `M`보다 커져버렸으면 `M`에 근접하도록 감소시키기 위해 현재의 구간에서 원소를 빼야 한다. 
    - ex) 1 [2 3 4] 5 6 👉 1 2 [3 4] 5 6
    - sum 에 arr[start] 를 빼주고 `start`를 증가시킨다.
    - `end`는 냅둔다. 
- 현재 구간의 합이 `M`과 일치한다면
  - 조건에 일치하는 구간이므로 카운팅한다! 
  - 현재 구간은 답이 되기 때문에 이 구간에서 앞으로 arr[start] 를 빼거나 arr[++end] 를 포함시켜도 답은 될 수 없을게 빤해서 그냥 둘 다 동시에 arr[start] 를 빼고 arr[++end] 를 더해주도록 한다. 
    - ex) 1 [2 3 4] 5 6 👉 1 2 [3 4 5] 6

<br>

#### ✈ 코드2 (카카오 인턴쉽 2020 : 보석 쇼핑)

> 두 포인터 모두 "오른쪽"으로 이동함

```cpp
#include <string>
#include <vector>
#include <algorithm>
#include <unordered_map>
#include <set>

using namespace std;

vector<int> solution(vector<string> gems) {
    vector<int> answer(2);
    set<string> kind;
    for (int i = 0; i < gems.size(); ++i)
        kind.insert(gems[i]); // set은 중복을 제거하므로 최종적으로 kind.size() 는 보석의 종류 개수가 된다.
    
    int minLen = 100000; // 구하는 구간
    unordered_map<string, int> count; // while문 반복 내에서 매 반복마다 해당 구간의 보석 종류 개수를 이 unordered_map 으로 셀 것이다. (Key 정렬 필요 없으므로 map 보단 더 빠른 unordered_map 채택)
    int i;
    int start = 0;
    int end = 0;
    while (true) {
        // end 포인터 업뎃 👉 보석의 모든 종류를 하나 이상씩 포함하는 곳까지
        for (i = end; i < gems.size(); ++i) {
            count[gems[i]]++;
            if (count.size() == kind.size()) {
                end = i;
                break;
            }
        }
        // end 포인터가 업뎃되지 못하고 빠져나와 gems 를 넘어섰다면 더 이상 업뎃할 구간 없으므로 종료
        if (i == gems.size())
            break;
        // start 포인터 업뎃 👉 보석의 모든 종류를 다 포함하되 이 보석을 제외하면 모든 종류를 포함할 수 없다싶은 곳 까지
        for (i = start; i < gems.size(); ++i) {
            if (count[gems[i]] == 1) {
                start = i;
                break;
            }
            else count[gems[i]]--;
        }
        // 현재 완성된 구간의 길이가 minLen 현재까지 구한 구간 최소 길이보다 작으면 minLen 업데이트
        if (end - start < minLen) {
            answer[0] = start + 1;
            answer[1] = end + 1;
            minLen = end - start;
        }
        // 이제 새로운 구간을 다음 반복부터 잡아야한다.
        // start, end 모두 1 씩 증가시킴 (start 와 end 의 다음 "후보" 시작점)
        // gems[start] 는 제외해야하므로 count 에서 제거 
        count.erase(gems[start]);
        start++;
        end++;
    }
    
    return answer;
}
```

> [[C++로 풀이] 보석 쇼핑 (투포인터 알고리즘)⭐⭐⭐](https://ansohxxn.github.io/programmers/118/)

![image](https://user-images.githubusercontent.com/42318591/111454172-b431d000-8757-11eb-91c4-c44af6f2f781.png)

- 모든 보석을 포함하되 최소로 포함하도록 구간의 start 와 end 를 확정짓는다. 
  - 현재의 start, end 가 가질 수 있는 값 내에서 찾는다.
- 현재 확정지은 구간이 현재까지 찾은 구간들 중 최소 길이가 될 수 있는지 검사한다.
- 다음 구간을 찾으러 `start++`, `end++`
{: .notice--warning}

> 보석의 종류 4 가지 👉 "RUBY", "DIA", "EMERALD", "SAPPHIRE"

1. 후보 <u>"DIA", "RUBY", "RUBY", "DIA", "EMERALD", "SAPPHIRE", "DIA"</u>
  - `end` 결정 👉 모든 다이아를 포함하도록 <u>"DIA", "RUBY", "RUBY", "DIA", "EMERALD", "SAPPHIRE"</u>, "DIA"
  - `start` 결정 👉 최소로 포함하도록 "DIA", "RUBY", <u>"RUBY", "DIA", "EMERALD", "SAPPHIRE"</u>, "DIA"
  - 현재의 구간 길이 4. `minLen`은 4 로 업데이트 (내가 end - start + 1 가 아닌 그냥 end - start 로 코딩했기 때문에 사실 3 이긴 하다)
2. 후보 "DIA", "RUBY", "RUBY", <u>"DIA", "EMERALD", "SAPPHIRE", "DIA"</u>
  - `end` 결정 👉 gems 의 끝에 도달할 때까지 모든 다이아를 포함하지 못해 종료! 

답은 4 가 된다.

<br>

### 🔥 포인터 2개가 양 끝에서 반대 방향으로 진행하는 경우

#### ✈ 코드 (백준 2470번 문제 : 두 용액)

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#define INF 2000000000
using namespace std;

vector<int> answer(2);
int main() {
    int N;
    cin >> N;
    vector<int> arr(N);
    for (int i = 0; i < N; ++i)
        cin >> arr[i];
    sort(arr.begin(), arr.end()); // 정렬 필수!

    int start = 0; // start 은 왼쪽 끝에서
    int end = N - 1; // end 은 오른쪽 끝에서 시작
    int min = INF; // 현재까지 0 에 가장 가까웠던 합
    while (start < end) {
        int sum = arr[start] + arr[end];

        if (min > abs(sum)) {
            min = abs(sum);
            answer[0] = arr[start];
            answer[1] = arr[end]; 

            if (sum == 0)
                break;
        }

        if (sum < 0) 
            start++;
        else 
            end--;
    }

    cout << answer[0] << " " << answer[1];
}
```

> [[백준 2470][💛5] 두 용액 (투포인터 알고리즘)](https://ansohxxn.github.io/boj/2470/)

양쪽 끝에서 출발한 두 포인터가 **서로 반대 방향**으로 다가와서 좁혀지는 형태의 투포인터 알고리즘이다.  start -> <- end 요렇게!

- `start` -> 방향
- `end` <- 방향

**서로의 합이 가장 0 에 가까운 두 수를 구해야하기 때문에 <u>먼저 오름차순 정렬을 한 후,</u>  <u>양쪽 끝에서부터 두 포인터를 이용하여 0 에 가까운 두 수를 구해나간다.</u>**
{: .notice--warning}

`min`은 현재까지 구한 두 수의 합 중 가장 0 에 가까웠던 합. (가장 "가까웠던" 이니까 절대값으로 넣어야 한다.)

- 1️⃣ 먼저 오름차순 정렬을 한다. (음수거나 작을 수록 왼쪽, 양수거나 클 수록 오른쪽에 위치하도록)
- 2️⃣ 양쪽 끝의 두 포인터가 가리키는 원소끼리 더하여 최대한 두 수의 합이 0 에 가깝도록 두 포인터를 업데이트 해나간다.
  - *arr[start] + arr[end] < 0* 
    - 0 보다 작으므로 두 수의 합이 더 커져야 0 에 가까워질 것이다. 그러므로 더 커져야하기 떄문에 `start`를 증가시킨다. (작은 값을 더 큰 값으로 대체)
  - *arr[start] + arr[end] > 0*
    - 0 보다 크므로 두 수의 합이 더 작아야 0 에 가까워질 것이다. 그러므로 더 작아져야하기 떄문에 `end`를 감소시킨다. (큰 값을 더 작은 값으로 대체)
- 3️⃣ `min` 업데이트 하기
  - *arr[start] + arr[end] < min* 
    - 기존에 구했던 두 수의 합보다 더 작은 합이 나왔으니 정답 후보가 될 수 있다. `min`과 `answer`를 업뎃한다. 
    - 혹시 이미 0 이 나왔으면 더 찾을 필요 없으므로 종료.

> 99 -2 -1 4 7

- 우선 정렬하기 👉 -99 -2 -1 4 7
- <u>-99</u> -2 -1 4 <u>7</u> : -99 + 7 < 0 이므로 더 커져야 함 (92)
- -99 <u>-2</u> -1 4 <u>7</u> : -2 + 7 > 0 이므로 더 작아져야 함 (5)
- -99 <u>-2</u> -1 <u>4</u> 7 : -2 + 4 > 0 이므로 더 작아져야 함 (2)
- -99 <u>-2</u> <u>-1</u> 4 7 : -2 + -1 < 0 이므로 더 커져야 함 (3)
  - start >= end 이므로 종료

답은 가장 0 에 가까웠던 (2)  -2  4 가 된다. 

<br>

### 🔥 포인터 하나는 한쪽 방향으로만 진행하고, 다른 포인터는 양쪽으로 이동하는 것

이 유형으로는 **모스 알고리즘(Mo's Algorithm)**이 있다고 한다. 기억해두었다가 나중에 관련 문제 접하게 되면 떠올리자.. ㅎㅎ

<br>

## 🚀 슬라이딩 윈도우

<iframe width="984" height="558" src="https://www.youtube.com/embed/uH9VJRIpIDY" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

> 어떤 특정 조건을 만족하는 **🔥연속 구간🔥**을 구할 때 `O(N)` 으로 풀 수 있도록 도와주는 알고리즘

어떤 창문을 왼쪽부터 오른쪽으로 밀면서 창문안에 들어오는 원소들이 바뀌는 모습을 상상해보면 된다. (창문은 고정적인 길이) 매순간 창문으로 보이는 모습의 정보 유출이 필요하다. (매 순간 창문안에 들어오는 원소들의 상태)

- 아래와 같이 고정적인 크기로 구간을 움직임 (크기 3이라고 가정)
  - [1,2,3],4,5 
  - 1,[2,3,4],5
  - 1,2,[3,4,5]

투포인터와 다른점이 있다면 투포인터 알고리즘은 구간의 길이를 가변적으로 잡기 때문에 구간의 양쪽 끝이 되는 포인터가 2 개 필요한 반면, 슬라이딩 윈도우 알고리즘은 **구간의 길이를 고정적으로 잡기 떄문에 포인터가 2 개일 필요가 없다.** 포인터 하나만 있어도 그 고정적인 길이를 알고 있으므로 끝이 어딘지 알 수 있기 때문.

구간의 고정 길이를(즉 창문의 너비를) L 이라고 할 떄, <u>창문을 한 칸 옮기더라도 옮기기 전 창문안에 있던 L - 1 칸은 겹치게 된다.</u> 

- 기존 모습 : 🟨🟪🟪🟪🟪🟨🟨
- 다음 모습 : 🟨🟨🟪🟪🟪🟪🟨

기존 구간과 새 구간의 🟨🟪🟥🟥🟥🟪🟨 빨간 부분은 겹치게 된다. 이미 🟥🟥🟥은 새 구간에도 포함이 되어 있으므로 새 구간에서는 빠지게 된 앞의 🟪 와 새 구간에 포함되게 된 🟪 만 비교하게 되면 더 효율적일 것이다. O(N^2)를 피할 수 있다. <u>이전의 결과를 써먹는 방향으로 접근하자.</u> 

- 슬라이딩 윈도우 문제 유형
  - 구간합 
    - 기존의 구간의 합이 S 이고 새로운 구간에선 빠지는 맨 앞 원소가 A 고 새로운 구간에 들어오게 된 원소를 B 라고 하면 새로운 구간의 합은 S - A + B 가 된다. 즉, 이 합의 과정이 `O(1)` 밖에 안되는 것이다. 일일이 새로운 구간의 합을 또 구할 필요가 없다. 따라서 전체적인 과정은 `O(N)` 이 된다. (맨 처음 창문의 합을 구할 때만 `O(W)`)
  - Anagram 찾기

### ✈ 코드 (백준 11003번 문제 : 최소값 찾기)

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <deque>
#include <queue>

using namespace std;

struct A {
    int data;
    int pos;
};

struct cmp {
    bool operator()(const A& a, const A& b) {
        return a.data > b.data;
    }
};

int main() {
    /* 입출력 속도 높이기 */
    ios::sync_with_stdio(false);
    cin.tie(NULL);
    cout.tie(NULL);

    int N, L;
    cin >> N >> L;
    vector<int> arr(N);
    for (int i = 0; i < N; ++i)
        cin >> arr[i];

    priority_queue<A, vector<A>, cmp> pq;
    for (int i = 0; i < N; ++i) {
        pq.push({ arr[i], i });
        int pos = pq.top().pos;
        while (pos < i - L + 1) {
            pq.pop();
            pos = pq.top().pos;
        }
        cout << pq.top().data << ' ';
    }
    return 0;
}
```

> [[백준 2470][💚5] 최소값 찾기 (슬라이딩 윈도우, deque)](https://ansohxxn.github.io/boj/11003/)

매번 새로운 창문 안에서의 최소값을 출력해야 하는 문제이다.

자세한 해설 풀이는 위 링크 참고! (deque 을 사용한 풀이, 우선순위 큐를 사용한 풀이 2 가지를 정리했다.)

> 1 5 2 3 6 2 3 7 3 5 2 6

[] 은 우선순위큐 안의 모습, 밑줄은 현재 구간 최소값, ~~은 pop 된 것~~

- i = 0 : [ <u>1(0)</u> ] 
- i = 1 : [ <u>1(0)</u>, 5(1) ]
- i = 2 : [ <u>1(0)</u>, 5(1), 2(2) ]
- i = 3 : ~~1(0)~~ [ 5(1), <u>2(2)</u>, 3(3) ]
  - 우선순위큐에서 가장 최소값(top)이 1(0) 인데 이는 범위를 벗어났으므로 pop
- i = 4 : [ 5(1), <u>2(2)</u>, 3(3), 6(4) ]
- i = 5 : ~~2(2)~~ [ 5(1), 3(3), 6(4), <u>2(5)</u> ]
  - 우선순위큐에서 가장 최소값(top)이 2(2) 인데 이는 범위를 벗어났으므로 pop
- i = 6 : [ 5(1), 3(3), 6(4), <u>2(5)</u>, 3(6) ]
- i = 7 : [ 5(1), 3(3), 6(4), <u>2(5)</u>, 3(6), 7(7) ]
- i = 8 : ~~2(5)~~ [ 5(1), 3(3), 6(4), <u>3(6)</u>, 7(7), 3(8) ]
  - 우선순위큐에서 가장 최소값(top)이 2(5) 인데 이는 범위를 벗어났으므로 pop
- i = 9 : [ 5(1), 3(3), 6(4), 3(6), 7(7), 3(8), 5(9) ]
- i = 10 : [ 5(1), 3(3), 6(4), 3(6), 7(7), 3(8), 5(9), <u>2(10)</u> ]
- i = 11 : [ 5(1), 3(3), 6(4), 3(6), 7(7), 3(8), 5(9), <u>2(10)</u>, 6(11) ]

<br>

## 🚀 Prefix Sum

- **Sum = Sum - A + B** 
- 혹은 누적합을 O(N) 으로 미리 구해놓은 후 (*Sum[i] = Sum[i - 1] + arr[i]*)
  - **Sum[i + A] - Sum[i]** 이런식

어떤 구간의 원소들을 구하는 합을 `Prefix Sum`으로 구하는 문제는 [[C++로 풀이] 광고 삽입 (구간합 Prefix Sum, 투포인터)⭐⭐⭐](https://ansohxxn.github.io/programmers/124/) 참고

***
<br>

    🌜 개인 공부 기록용 블로그입니다. 오류나 틀린 부분이 있을 경우 
    언제든지 댓글 혹은 메일로 지적해주시면 감사하겠습니다! 😄

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}