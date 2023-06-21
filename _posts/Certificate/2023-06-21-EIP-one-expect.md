---
title:  "[정보처리기사] 1과목 : 소프트웨어 설계 예상문제 " 

categories:
  - Certificate
tags:
  - [Algorithm]

toc: true
toc_sticky: true

date: 2023-06-21
last_modified_at: 2023-06-21
---

# 정보처리기사

## 1과목 : 소프트웨어 설계

### 예상문제은행

#### <span style="color:#0489B1;">1. 소프트웨어 생명주기 모델 중 아래 보기가 설명하는 모형은?</span>
```
a. 고객과의 의사소통을 통해 계획 수립과 위험 분석, 구축, 고객 평가의 과정을 거쳐 소프트웨어를 개발한다.
b. 가장 큰 장점인 위험 분석 단계에서 기술과 관리의 위험요소들을 하나씩 제거해 나감으로써 완성도 높은 소프트웨어를 만들 수 있다.
c. 반복적인 작업을 수행하는 점증적 생명 주기 모델이다.
d. 비용이 많이 들거나 시간이 많이 소요되는 대규모 프로젝트나 큰 시스템을 구축할 때 유리하다.
```
1. 프로토타입 모델
2. 폭포수 모델
3. 나선형 모델
4. RAD 모델

<details>
<summary>정답 및 해설보기</summary>

<blockquote>
<p>정답 : 3번</p>
<p>풀이 : 계획수립, 위험분석, 구축, 고객평가 과정을 거치는 모형은 나선형 모델이다.</p>
</blockquote>
<hr/>
</details>

<details>
<summary>문제에 사용된 내용보기</summary>

<blockquote>
  <p><strong>나선형 모형</strong></p>
<ul>
  <li>보헴이라는 사람이 제안한 모델로 폭포수 모형과 프로토타입 모형의 장점에 <u>위험 분석기능</u>을 추가한 모형</li>
  <li>나선을 따라 돌듯이 여러 번의 소프트웨어 개발 과정을 거쳐 점진적으로 완벽한 최종 소프트웨어를 개발하는 것으로 점진적 모형이라고도 한다.</li>
  <li>소프트웨어를 개발하면서 발생할 수 있는 위험을 관리하고 최소화 하는 것을 목적으로 한다.</li>
  <li>점전적으로 개발 과정이 반복되므로 누락되거나 추가된 요구사항을 첨가할 수 있고, 정밀하며, 유지보수 과정이 필요없다.</li>
  <li>비용이 많이 들거나 시간이 많이 소요되는 대규모 프로젝트나 큰 시스템을 구축할 때 유리하다.</li>
</ul>
  <p>계획 수립 -> 위험 분석 -> 개발 및 검증 -> 고객 평가 를 반복한다. </p>
  <span style="color:red">계위개고</span>
</blockquote>
<hr/>
</details>

#### <span style="color:#0489B1;">2. 소프트웨어 공학 패러다임에 해당하지 않는것은?</span>
1. 폭포수 모형
2. 프로토타입 모형
3. 나선형 모형
4. 3세대 기법

<details>
<summary>정답 및 해설보기</summary>

<blockquote>
<p>정답 : 4번</p>
<p>풀이 : 3세대 기법은 소프트웨어 공학 패러다임에 속하지 않는다.</p>
</blockquote>
<hr/>
</details>

<details>
<summary>문제에 사용된 내용보기</summary>

<blockquote>
  <p><strong>소프트웨어 공학 패러다임</strong></p>
<ul>
  <li>폭포수 모형(Waterfall Model)</li>
  <li>프로토타입 모형(Prototype Model)</li>
  <li>나선형 모형(Spiral Model)</li>
  <li>4세대 기법(4GT)</li>
  <li>애자일 모형(Agile Model)</li>
</ul>
</blockquote>
<hr/>
</details>


#### <span style="color:#0489B1;">3. 소프트웨어 공학에서 가장 폭넓게 사용되고 있는 모형은?</span>
1. 폭포수 모형
2. 프로토타입 모형
3. 나선형 모형
4. 4세대 기법

<details>
<summary>정답 및 해설보기</summary>

<blockquote>
<p>정답 : 1번</p>
<p>풀이 : 가장 오래되고, 가장 폭넓게 사용된 모형은 폭포수 모형이다.</p>
</blockquote>
<hr/>
</details>

<details>
<summary>문제에 사용된 내용보기</summary>

<blockquote>
  <p><strong>폭포수 모형</strong></p>
<ul>
  <li>폭포수 모형은 소프트웨어 공학에서 가장 오래되고 폭넓게 사용된 전통적인 소프트웨어 생명 주기 모형으로, <u>고전적 생명 주기 모형</u>이라고도 한다.</li>
  <li>소프트웨어 개발 과정의 한 단계가 끝나야만 다음 단계로 넘어갈 수 있는 <u>선형 순차적 모형</u>이다.</li>
  <li>모형을 적용한 경험과 성공 사례가 많다.</li>
  <li>제품의 일부가 될 메뉴얼을 작성해야 한다.</li>
  <li>각 단계가 끝난 후에는 다음 단계를 수행하기 위한 결과물이 명확하게 산출되어야 한다.</li>
  <li>그러므로 개발 중간에 <u>요구사항의 변경이 용이하지 않다.</u></li>
</ul>
  <p>타당성 검토 -> 계획 -> 요구 분석 -> 설계 -> 구현 -> 테스트(검사) -> 유지보수</p>
  <span style="color:red">분설구테유</span>
</blockquote>
<hr/>
</details>


#### <span style="color:#0489B1;">4. 다음중 소프트웨어 개발 모형이 가장 적절하게 선택된 경우는?</span>
1. 구축하고자 하는 시스템의 요구사항이 불분명하여 프로토타입을 선택하였다.
2. 개발 중에도 고객의 요구사항에 맞게 수정작업을 할 수 있도록 폭포수 모형을 선택하였다.
3. 위험 분석을 통해 점증적으로 시스템을 개발할 수 있도록 폭포수 모형을 선택하였다.
4. 응용 분야가 단순하고 설치 시점에 제품 설명서가 요구됨에 따라 나선형 모형을 선택하였다.

<details>
<summary>정답 및 해설보기</summary>

<blockquote>
<p>정답 : 1번</p>
<p>풀이 : 요구사항이 불분명할 때 분명하게 하기위해 프로토 모델을 사용한다.</p>
</blockquote>
<hr/>
</details>

<details>
<summary>문제에 사용된 내용보기</summary>

<blockquote>
<ul>
  <li>2. 개발 중에도 고객의 요구사항에 맞게 수정작업을 할 수 있도록 하려면 <s>폭포수 모형</s>이 아닌 프로토 타입을 사용해야한다.</li>
  <li>3. 위험 분석을 통해 점증적으로 시스템을 개발할 수 있도록 <s>폭포수 모형</s>이 아닌 나선형 모델을 사용해야한다.</li>
  <li>4. 응용 분야가 단순하고 설치 시점에 제품 설명서가 요구됨에 따라 <s>나선형 모델</s>이 아닌 폭포수 모형을 사용해야한다.</li>
</ul>
</blockquote>
<hr/>
</details>


#### <span style="color:#0489B1;">5. 프로토타입 모델 개발 방법이 가장 적절하게 적용될 수 있는 경우는?</span>
1. 테스트 작업이 중요하지 않을 경우
2. 고객이 빠른 시간 내에 개발의 완료를 요구할 경우
3. 구축하고자 하는 시스템의 요구사항이 불명확한 경우 
4. 고객이 개발 과정에는 참여하지 않고자 하는 경우

<details>
<summary>정답 및 해설보기</summary>

<blockquote>
<p>정답 : 3번</p>
<p>풀이 : 요구사항이 불분명할 때 분명하게 하기위해 프로토 모델을 사용한다.</p>
</blockquote>
<hr/>
</details>

<details>
<summary>문제에 사용된 내용보기</summary>

<blockquote>
  <p><strong>프로토타입 모형</strong></p>
<ul>
  <li>견본품을 만들어 최종 결과물을 예측하는 모형이다.</li>
  <li>사용자와 시스템 사이의 인터페이스에 중점을 두어 개발한다.</li>
  <li><u>개발 중간에 요구사항의 변경이 용이</u>하다.</li>
</ul>
</blockquote>
<hr/>
</details>


#### <span style="color:#0489B1;">6. 고전적 생명 주기 모델에 프로토타입 모형의 장점과 위험 분석 기능을 추가한 패러다임은?</span>
1. Waterfall Model
2. Spiral Model
3. Jackson Model
4. 4GT

<details>
<summary>정답 및 해설보기</summary>

<blockquote>
<p>정답 : 2번</p>
<p>풀이 : 나선형 모델</p>
</blockquote>
<hr/>
</details>

<details>
<summary>문제에 사용된 내용보기</summary>

<blockquote>
<p>영어로 시험 출제가 나오기도 하니 영어로도 알아두면 좋을듯 하다.</p>
<ul>
  <li>폭포수 모형 : Waterfall Model</li>
  <li>프로토타입 모형 : Prototype Model</li>
  <li>나선형 모형 : Spiral Model</li>
  <li>애자일 모형 : Agile Model</li>
  <li>4세대 기법 : 4GT(4th Generation Language)</li>
</ul>
</blockquote>
<hr/>
</details>


#### <span style="color:#0489B1;">7. 소프트웨어 생명 주기 모형 중 Spiral Model에 대한 설명으로 옳지 않은 것은?</span>
1. 대규모 시스템에 적합하다.
2. 초기에 위험 요소를 발견하지 못할 경우 위험 요소를 제거하기 위해 많은 비용이 소요될 수 있다.
3. 소프트웨어를 개발하면서 발생할 수 있는 위험을 고나리하고 최소화 하는 것을 목적으로 한다.
4. 소프트웨어 개발 과정의 앞 단계가 끝나야만 다음 단계로 넘어갈 수 있는 선형 순차적 모형이다.

<details>
<summary>정답 및 해설보기</summary>

<blockquote>
<p>정답 : 4번</p>
<p>풀이 : 개발 과정의 앞 단계가 끝나야만 다음 단계로 넘어갈 수 있는 선형 순차적 모형은 폭포수 모형이다.</p>
</blockquote>
<hr/>
</details>

<details>
<summary>문제에 사용된 내용보기</summary>

<blockquote>
  <p><strong>나선형 모형</strong></p>
<ul>
  <li>보헴이라는 사람이 제안한 모델로 폭포수 모형과 프로토타입 모형의 장점에 <u>위험 분석기능</u>을 추가한 모형</li>
  <li>나선을 따라 돌듯이 여러 번의 소프트웨어 개발 과정을 거쳐 점진적으로 완벽한 최종 소프트웨어를 개발하는 것으로 점진적 모형이라고도 한다.</li>
  <li>소프트웨어를 개발하면서 발생할 수 있는 위험을 관리하고 최소화 하는 것을 목적으로 한다.</li>
  <li>점전적으로 개발 과정이 반복되므로 누락되거나 추가된 요구사항을 첨가할 수 있고, 정밀하며, 유지보수 과정이 필요없다.</li>
  <li>비용이 많이 들거나 시간이 많이 소요되는 대규모 프로젝트나 큰 시스템을 구축할 때 유리하다.</li>
</ul>
  <p>계획 수립 -> 위험 분석 -> 개발 및 검증 -> 고객 평가 를 반복한다. </p>
  <span style="color:red">계위개고</span>
</blockquote>
<hr/>
</details>


#### <span style="color:#0489B1;">8. 소프트웨어 생명 주기 모형 중 프로토타입 모형의 가장 큰 장점이라고 볼 수 있는 것은?</span>
1. 개발 비용의 절감
2. 4세대 언어의 적용
3. 개발 단계의 명확성
4. 요구사항의 정확한 파악

<details>
<summary>정답 및 해설보기</summary>

<blockquote>
<p>정답 : 4번</p>
<p>풀이 : 고객의 요구사항이 불분명할때 요구사항을 정확히 파악하기 위해 프로토타입을 사용한다.</p>
</blockquote>
<hr/>
</details>

<details>
<summary>문제에 사용된 내용보기</summary>

<blockquote>
  <p><strong>프로토타입 모형</strong></p>
<ul>
  <li>견본품을 만들어 최종 결과물을 예측하는 모형이다.</li>
  <li>사용자와 시스템 사이의 인터페이스에 중점을 두어 개발한다.</li>
  <li><u>개발 중간에 요구사항의 변경이 용이</u>하다.</li>
</ul>
</blockquote>
<hr/>
</details>


#### <span style="color:#0489B1;">9.소프트웨어 개발 모형 중 나선형 모델의 활동 과정이 아닌 것은?</span>
1. 계획 및 정의
2. 위험 분석
3. 개발
4. 유지보수

<details>
<summary>정답 및 해설보기</summary>

<blockquote>
<p>정답 : 4번</p>
<p>풀이 : 계획 수립 -> 위험 분석 -> 개발 및 검증 -> 고객 평가를 반복한다. <strong style="color:red;">계위개고</strong></p>
</blockquote>
<hr/>
</details>

<details>
<summary>문제에 사용된 내용보기</summary>

<blockquote>
  <p><strong>나선형 모형</strong></p>
<p>계획 수립 -> 위험 분석 -> 개발 및 검증 -> 고객 평가를 반복한다.<br>
나선형 모형관련하여 과정 관련 문제가 종종 출제되는듯함. </p>
</blockquote>
<hr/>
</details>

#### <span style="color:#0489B1;">10. 여러 번의 개발 과정을 거쳐 완벽한 최종 소프트웨어를 개발하는 점진적 모형으로 보헴이 제안한 소프트웨어 생명주기 모델은?</span>
1. 4GT Model
2. Spiral Model
3. Waterfall Model
4. Prototype Model

<details>
<summary>정답 및 해설보기</summary>

<blockquote>
<p>정답 : 2번</p>
<p>풀이 : 계획 수립 -> 위험 분석 -> 개발 및 검증 -> 고객 평가를 반복해가며 개발한다.
또한, 점진적 모형을 말하면 나선형으로 생각</p>
</blockquote>
<hr/>
</details>

<details>
<summary>문제에 사용된 내용보기</summary>

<blockquote>
  <p><strong>나선형 모형</strong></p>
<ul>
  <li>보헴이라는 사람이 제안한 모델로 폭포수 모형과 프로토타입 모형의 장점에 <u>위험 분석기능</u>을 추가한 모형</li>
  <li>나선을 따라 돌듯이 여러 번의 소프트웨어 개발 과정을 거쳐 점진적으로 완벽한 최종 소프트웨어를 개발하는 것으로 점진적 모형이라고도 한다.</li>
  <li>소프트웨어를 개발하면서 발생할 수 있는 위험을 관리하고 최소화 하는 것을 목적으로 한다.</li>
  <li>점전적으로 개발 과정이 반복되므로 누락되거나 추가된 요구사항을 첨가할 수 있고, 정밀하며, 유지보수 과정이 필요없다.</li>
  <li>비용이 많이 들거나 시간이 많이 소요되는 대규모 프로젝트나 큰 시스템을 구축할 때 유리하다.</li>
</ul>
  <p>계획 수립 -> 위험 분석 -> 개발 및 검증 -> 고객 평가 를 반복한다. </p>
  <span style="color:red">계위개고</span>
</blockquote>
<hr/>
</details>


#### <span style="color:#0489B1;">11. 소프트웨어 공학의 전통적인 개발 방법인 선형 순차 모형의 순서를 옳게 나열한 것은?</span>
1. 구현 -> 분석 -> 설계 -> 테스트 -> 유지보수
2. 유지보수 -> 테스트 -> 분석 -> 설계 -> 구현
3. 분석 -> 설계 -> 구현 -> 테스트 -> 유지보수
4. 테스트 -> 설계 -> 유지보수 -> 구현 -> 분석

<details>
<summary>정답 및 해설보기</summary>

<blockquote>
<p>정답 : 3번</p>
<p>풀이 : 전통적인 개발 방법은 폭포수 모형을 말하며 순서는 분석 -> 설계 -> 구현 -> 테스트 -> 유지보수 <span style="color:red;">분설구테유</span> </p>
</blockquote>
<hr/>
</details>

<details>
<summary>문제에 사용된 내용보기</summary>

<blockquote>
  <p><strong>폭포수 모형</strong></p>
<ul>
  <li>폭포수 모형은 소프트웨어 공학에서 가장 오래되고 폭넓게 사용된 전통적인 소프트웨어 생명 주기 모형으로, <u>고전적 생명 주기 모형</u>이라고도 한다.</li>
  <li>소프트웨어 개발 과정의 한 단계가 끝나야만 다음 단계로 넘어갈 수 있는 <u>선형 순차적 모형</u>이다.</li>
  <li>모형을 적용한 경험과 성공 사례가 많다.</li>
  <li>제품의 일부가 될 메뉴얼을 작성해야 한다.</li>
  <li>각 단계가 끝난 후에는 다음 단계를 수행하기 위한 결과물이 명확하게 산출되어야 한다.</li>
  <li>그러므로 개발 중간에 <u>요구사항의 변경이 용이하지 않다.</u></li>
</ul>
  <p>타당성 검토 -> 계획 -> 요구 분석 -> 설계 -> 구현 -> 테스트(검사) -> 유지보수</p>
  <span style="color:red">분설구테유</span>
</blockquote>
<hr/>
</details>


#### <span style="color:#0489B1;">12. 폭포수 모델에 대한 설명으로 옳지 않은 것은?</span>
1. 소프트웨어 개발 과정의 각 단계가 순차적으로 진행된다.
2. 앞 단계에서 발견하지 못한 오류를 다음 단계에서 발견했을 때 오류 수정이 용이하다.
3. 두 개 이상의 과정이 병행 수행되거나 이전 단계로 넘어가는 경우가 없다.
4. 개발 과정중에 발생하는 새로운 요구나 경험을 설계에 반영하기 어렵다.

<details>
<summary>정답 및 해설보기</summary>

<blockquote>
<p>정답 : 2번</p>
<p>풀이 : 폭포수 모델은 한 번 지나간곳을 다시 거슬러 올라갈 수 없는 구조로 이전 단계의 오류 수정에 어려움이 있다.</p>
</blockquote>
<hr/>
</details>

<details>
<summary>문제에 사용된 내용보기</summary>

<blockquote>
  <p><strong>폭포수 모형</strong></p>
<ul>
  <li>폭포수 모형은 소프트웨어 공학에서 가장 오래되고 폭넓게 사용된 전통적인 소프트웨어 생명 주기 모형으로, <u>고전적 생명 주기 모형</u>이라고도 한다.</li>
  <li>소프트웨어 개발 과정의 한 단계가 끝나야만 다음 단계로 넘어갈 수 있는 <u>선형 순차적 모형</u>이다.</li>
  <li>모형을 적용한 경험과 성공 사례가 많다.</li>
  <li>제품의 일부가 될 메뉴얼을 작성해야 한다.</li>
  <li>각 단계가 끝난 후에는 다음 단계를 수행하기 위한 결과물이 명확하게 산출되어야 한다.</li>
  <li>그러므로 개발 중간에 <u>요구사항의 변경이 용이하지 않다.</u></li>
</ul>
  <p>타당성 검토 -> 계획 -> 요구 분석 -> 설계 -> 구현 -> 테스트(검사) -> 유지보수</p>
  <span style="color:red">분설구테유</span>
</blockquote>
<hr/>
</details>

#### <span style="color:#0489B1;">13. 다음의 특징에 맞는 개발 접근방식은?</span>
```
- 유용한 소프트웨어를 빠르고, 지속적으로 제공하여 고객을 만족시킨다.
- 개발 막바지에서 요구사항 변경을 환영한다.
- 같은 사무실에서 얼굴을 맞대고 의견을 나눈다.
- 동기가 부여된 개발자로 팀을 구성하고, 신뢰하고, 개발환경을 제공하고 지원한다.
```
1. 컴포넌트 기반 개발 (Component Based Development)
2. 정보 공학 개발 (Information Engineering Development)
3. 애자일 개발 (Agile Programming)
4. 객체 지향 개발 (Object Orient Development)

<details>
<summary>정답 및 해설보기</summary>

<blockquote>
<p>정답 : 3번</p>
<p>풀이 : 애자일 기법에 대한 설명이다.</p>
</blockquote>
<hr/>
</details>

<details>
<summary>문제에 사용된 내용보기</summary>

<blockquote>
  <p><strong>애자일 모형</strong></p>
<ul>
  <li>애자일 모형은 어느 특정 개발 방법론이 아니라 상황에 따라 좋은 것을 빠르고 낭비 없게 만들기 위해 <u>고객과의 소통에 초점을 맞춘 방법론을 통칭</u>한다.</li>
  <li><u>스프린트 또는 이터레이션이라고 불리는 짧은 개발 주기를 반복하면서 개발과정을 진행</u>하며, 주기마다 고객의 평과와 요구를 수용한다.</li>
  <li>각 개발주기에서 고객이 요구사항에 우선순위를 부여하여 작업을 진행한다.</li>
  <li>고객과 소통에 초점을 둔 방법으로 <u>변화에 유연하게 대응이 가능하다.</u></li>
</ul>
<p>애자일 모형에는 XP(eXtreme Programming), 스크럼(Scrum), 칸반(Kanban), 크리스탈(Crystal), 린(LEAN), 기능중심개발(FDD), DSDM, DAD 등이 있다.</p>
<span style="color:red;">엑스칸크린</span> 로 외워두면 좋을듯 하다. (나머지는 선택)
</blockquote>
<hr/>
</details>


#### <span style="color:#0489B1;">14. 폭포수 모형과 애자일을 비교했을 때 애자일의 특징이 아닌것은?</span>
1. 새로운 요구사항의 반영이 쉽다.
2. 개발에 있어 계획보다는 고객을 중심으로 한다.
3. 개발이 완료되면 최종적으로 모든 기능을 테스트한다.
4. 지속적으로 고객과 의사소통을 수행한다.

<details>
<summary>정답 및 해설보기</summary>

<blockquote>
<p>정답 : 3번</p>
<p>풀이 : 애자일은 짧은 개발 주기를 반복하며, 반복되는 주기마다 만들어지는 결과물에 대한 고객의 평가와 요구를 적극
수용한다. 개발이 완료되면 최종적으로 모든 기능을 테스트하는 것은 폭포수의 특징이다.</p>
</blockquote>
<hr/>
</details>

<details>
<summary>문제에 사용된 내용보기</summary>

<blockquote>
  <p><strong>애자일 모형</strong></p>
<ul>
  <li>애자일 모형은 어느 특정 개발 방법론이 아니라 상황에 따라 좋은 것을 빠르고 낭비 없게 만들기 위해 <u>고객과의 소통에 초점을 맞춘 방법론을 통칭</u>한다.</li>
  <li><u>스프린트 또는 이터레이션이라고 불리는 짧은 개발 주기를 반복하면서 개발과정을 진행</u>하며, 주기마다 고객의 평과와 요구를 수용한다.</li>
  <li>각 개발주기에서 고객이 요구사항에 우선순위를 부여하여 작업을 진행한다.</li>
  <li>고객과 소통에 초점을 둔 방법으로 <u>변화에 유연하게 대응이 가능하다.</u></li>
</ul>
<p>애자일 모형에는 XP(eXtreme Programming), 스크럼(Scrum), 칸반(Kanban), 크리스탈(Crystal), 린(LEAN), 기능중심개발(FDD), DSDM, DAD 등이 있다.</p>
<span style="color:red;">엑스칸크린</span> 로 외워두면 좋을듯 하다. (나머지는 선택)
</blockquote>
<hr/>
</details>


#### <span style="color:#0489B1;">15. 다음 중 스크럼에 대한 설명으로 잘못된 것은?</span>
1. 제품 개발에 필요한 모든 요구사항을 우선순위에 따라 나열한 제품 백로그를 사용한다.
2. 소멸차트를 통해 작업의 진행 상황을 확인할 수 있다.
3. 스프린트 검토 회의에서 개선할 사항에 대한 피드백이 정리되면 스크럼 마스터는 이를 다음 스프린트에 반영할 수 있도록 제품 백로그를 업데이트한다.
4. 스프린트 동안 진행될 작업들을 개발자별로 할당할 때는 개발자들이 자신에게 맞는 직업을 스스로 선별하여 담당할 수 있도록 하는 것이 좋다.

<details>
<summary>정답 및 해설보기</summary>

<blockquote>
<p>정답 : 3번</p>
<p>풀이 : 스프린트 검토 회의에서 개선할 사항에 대한 피드백이 정리되면 제품책임자(PO)는 이를 다음 스프린트에 반영할 수 있도록 제품 백로그를 업데이트한다.</p>
</blockquote>
<hr/>
</details>

<details>
<summary>문제에 사용된 내용보기</summary>

<blockquote>
  <p><strong>스크럼 기법</strong></p>
<ul>
  <li>개발 작업에 관한 모든 것을 스스로 해결해야 함</li>
  <li>스프린트는 2~4주 정도의 기간으로 진행</li>
  <li>게획 회의 -> 스프린트 -> 일일 스크럼 -> 스크럼 검토 회의 -> 스프린트 회고  <span style="color:red;"> 계스일검회</span></li>
  <li>추가 내용은 따로 확인...</li>
</ul>
</blockquote>
<hr/>
</details>


#### <span style="color:#0489B1;">16. 다음 제시된 XP(eXtream Programming)의 개발 프로세스를 순서에 맞게 나열한 것은?</span>
``` 
ㄱ. 주기 
ㄴ. 승인 검사
ㄷ. 릴리즈 계획 수립
ㄹ. 소규모 릴리즈
```
1. ㄱ - ㄴ - ㄷ - ㄹ
2. ㄴ - ㄷ - ㄹ - ㄱ
3. ㄷ - ㄴ - ㄹ - ㄱ
4. ㄷ - ㄱ - ㄴ - ㄹ

<details>
<summary>정답 및 해설보기</summary>

<blockquote>
<p>정답 : 4번</p>
<p>풀이 : 릴리즈 계획 수립 -> 주기 -> 승인 검사 -> 소규모 릴리즈</p>
</blockquote>
<hr/>
</details>

<details>
<summary>문제에 사용된 내용보기</summary>

<blockquote>
  <p><strong>XP의 핵심 가치</strong></p>
<ul>
  <li>용기(Courage)</li>
  <li>단순성(Simplicity)</li>
  <li>의사소통(Communication)</li>
  <li>피드백(Feedback)</li>
  <li>존중(Respect)</li>
</ul>
<span style="color:red;">용단의피존</span>
<p><strong>XP의 기본원리</strong></p>
<ul>
  <li>전체팀(Whole Team)</li>
  <li>소규모 릴리즈(Small Releases)</li>
  <li>테스트 주도 개발(Test-Driven Development)</li>
  <li>계속적인 통합(Continuous Integration)</li>
  <li>공동 소유권(Collective Ownership)</li>
  <li>짝 프로그래밍(Pair Programming)</li>
  <li>디자인 개선(Design Improvement) 또는 리팩토링(Refactoring)</li>
</ul>
<span style="color:red;">전소테 계공짝디</span>
</blockquote>
<hr/>
</details>


#### <span style="color:#0489B1;">17. 다음중 오픈 소스 사용에 따른 고려사항에 속하지 않는 것은?</span>
1. 라이선스의 종류
2. 사용자 수
3. 기술의 지속 가능성
4. 라이선스의 비용

<details>
<summary>정답 및 해설보기</summary>

<blockquote>
<p>정답 : 4번</p>
<p>풀이 : 오픈 소스란 누구나 제한 없이 무료로 사용할 수 있도록 소스 코드를 공개한 것이다.</p>
</blockquote>
<hr/>
</details>

<details>
<summary>문제에 사용된 내용보기</summary>

<blockquote>
  <p><strong>오픈소스</strong></p>
<p>오픈소스는 OPEN SOURCE 무료로 사용하는 자원이다.</p>
</blockquote>
<hr/>
</details>


#### <span style="color:#0489B1;">18. 사용자 요구사항 추출 방법 중 시스템 수행 결과를 설명하기 위해 종이에 화면 순서를 기술하여 고객과 사용자에게 보여주는 것과 관련된 것은?</span>
1. 인터뷰
2. 설문
3. 브레인스토밍
4. 프로토타이핑

<details>
<summary>정답 및 해설보기</summary>

<blockquote>
<p>정답 : 4번</p>
<p>풀이 : 문제가 설명</p>
</blockquote>
<hr/>
</details>

<details>
<summary>문제에 사용된 내용보기</summary>

<blockquote>
<ul>
  <li>인터뷰 : 상대를 직접 만나서 이야기하면서 의견을 나누는 방법</li>
  <li>설문 : 어떤 주제에 대하여 미리 준비한 질문을 내어 물어보는 방법</li>
  <li>브레인스토밍 : 3인 이상이 자유롭게 의견을 교환하면서 독창적인 아이디어를 산출해 내는 방법</li>
</ul>
</blockquote>
<hr/>
</details>


#### <span style="color:#0489B1;">19. 요구 분석에 대한 설명으로 옳지 않은 것은?</span>
1. 고객이나 개발자 누구나 알 수 있는 내용은 요구사항에서 생략하여도 무방하다.
2. 요구사항은 소프트웨어를 개발하고 검증하는 기반이된다.
3. 요구사항은 명확하고 구체적이며 검증이 가능해야한다.
4. 요구사항은 크게 기능적인 요구사항과 비기능적 요구사항으로 분류된다.

<details>
<summary>정답 및 해설보기</summary>

<blockquote>
<p>정답 : 1번</p>
<p>풀이 : 요구사항은 제품 개발 과정에서 검증 테스트를 위한 기반되므로 개발에 필요한 모든 요소가 빠짐없이 완전하게 기술되어야 한다.</p>
</blockquote>
<hr/>
</details>

<details>
<summary>문제에 사용된 내용보기</summary>

<blockquote>
<ul>
  <li>요구사항은 소프트웨어 개발이나 유지 보수 과정에서 필요한 기준과 근거를 제공한다.</li>
  <li>요구사항은 개발하려는 소프트웨어의 전반적인 내용을 확인할 수 있게 하므로 개발에 참여하는 이해관계자들 간의 의사소통을 원활하게 하는 데 도움을 준다.</li>
  <li>요구사항이 제대로 정의되어야만 이를 토대로 이후 과정의 목표와 계획을 수립할 수 있다.</li>
</ul>
</blockquote>
<hr/>
</details>


#### <span style="color:#0489B1;">20. 다음은 서점 시스템의 요구사항에 대한 내용이다. 비기능 요구사항에 대한 설명은?</span>
1. 사용자는 로그인 또는 비로그인을 통해 책을 구매할 수 있어야 한다.
2. 사용자가 책을 현금으로 구매하였을 경우 현금영수증 처리를 할 수 있어야 한다.
3. 동시에 100명 이상이 주문을 요청해도 처리할 수 있어야 한다.
4. 사용자가 마이페이지에 저장해 놓은 도서 목록은 일정기간 동안 그대로 저장되어 있어야 한다.

<details>
<summary>정답 및 해설보기</summary>

<blockquote>
<p>정답 : 3번</p>
<p>풀이 : 시스템의 품질에 관한 요구사항으로 비기능 요구사항에 해당한다.</p>
</blockquote>
<hr/>
</details>

<details>
<summary>문제에 사용된 내용보기</summary>

<blockquote>
<p><strong>기능 요구사항과 비기능 요구사항</strong></p>
<ul>
  <li>기능 요구사항 : 기능, 입력, 출력, 저장, 수행 등...</li>
  <li>비기능 요구사항 : 성능, 품질, 제약사항, 호환성 보안 등...</li>
</ul>
</blockquote>
<hr/>
</details>


#### <span style="color:#0489B1;">21. 요구사항 협상은 요구사항이 서로 충돌하는 경우 이를 적절히 해결하는 과정이다. 다음 중 요구사항이 서로 충돌되어 적절한 기준점을 찾아 합의해야 하는 경우가 아닌 것은?</span>
1. 두 명의 이해관계자가 요구하는 요구사항이 서로 충돌되는 경우
2. 요구사항이 서로 우선순위가 다른 경우
3. 요구사항과 자원이 서로 충돌되는 경우
4. 기능 요구사항과 비기능 요구사항이 서로 충돌되는 경우

<details>
<summary>정답 및 해설보기</summary>

<blockquote>
<p>정답 : 2번</p>
<p>풀이 : 이미 우선순위가 부여된 요구사항은 충돌이 발생하지 않는다.</p>
</blockquote>
<hr/>
</details>

<details>
<summary>문제에 사용된 내용보기</summary>

<blockquote>
<p><strong>기능 요구사항과 비기능 요구사항</strong></p>
<ul>
  <li>기능 요구사항 : 기능, 입력, 출력, 저장, 수행 등...</li>
  <li>비기능 요구사항 : 성능, 품질, 제약사항, 호환성 보안 등...</li>
</ul>
</blockquote>
<hr/>
</details>






*** 

<br>

    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}