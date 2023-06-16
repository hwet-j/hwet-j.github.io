---
title:  "[Java] 여러가지 기능"  
excerpt: "어러가지 라이브러리, 메서드, 정규표현식 등을 알아보자"

categories: # 분류하고싶은 카테고리 입력
  - Java
tags:     # 중요 키워드
  - [Term, Function, Method, Library]

toc: true
toc_sticky: true
published: true

author: Hwet

date: 2023-06-16
last_modified_at: 2023-06-16
---

인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1]()
{: .notice--warning}

## 잡다 기능

### 대,소문자로 변경하기

문자열의 경우에는 toUpperCase(), toLowerCase()를 사용하여 쉽게 변경가능하다.

> 사용 예시

```java
String str = "asndFaDAna";

System.out.println(str.toLowerCase()); // -> asndfadana
System.out.println(str.toUpperCase()); // -> ASNDFADANA
```

### 특수문자

##### 1. 이스케이프 문자를 사용하여 특수문자 출력

- 개행 (줄바꿈): \n
- 탭: \t
- 작은따옴표: '
- 큰따옴표: "
- 백슬래시 자체: \

```java 
System.out.println("안녕\\하세요!\n\"특수문자\"를\t\'알아\'보고있습니다.");
```

위의 코드는 아래와 같이 출력됩니다.

```
안녕\하세요!
"특수문자"를	'알아'보고있습니다.
```

##### 2. 특수문자 제거하기

> ❗ 특수문자를 제거하기 전에 정규 표현식에 대해 알아보자.

정규 표현식(Regular Expression)은 문자열의 패턴을 기술하는 표현식이다.

문자열 검색, 추출, 대체 등의 작업을 수행하는데 사용되며, 자바 뿐만 아니라 다양한 프로그래밍 언어에서 지원한다.

1. . (마침표)
  - 어떤 한 개의 문자를 나타냅니다.
  - 즉, 모든 문자를 말합니다.
  - EX) "a.b." 는 "abbb", "aybq", "acbw" 와 동일합니다.  
2. \* (별표): 앞의 문자가 0개 반복됨을 나타냅니다.
   - 앞에있는 문자는 다른 문자가 나올때까지 없는 것으로 간주한다는 의미이다.
   - EX) "a*b" 는 "b", "aaaaab", "aaaaaaaab" 와 동일합니다. 
3. \+ (더하기): 앞의 문자가 1개 이상 반복됨을 나타냅니다.
  - 별표와 비슷하지만 별표는 이어진 문자를 0개로 간주하지만 더하기는 1개로 간주합니다.
  - EX) "a+b" 는 "ab", "aaaaab", "aaaaaaaab" 와 동일합니다.
4. ? (물음표): 앞의 문자가 0개 또는 1개 등장함을 나타냅니다.
  - EX) "a?b"는 "b", "ab"와 일치하지만 "aab"와는 일치하지 않습니다.
5. [] (대괄호): 대괄호 안에 나열된 문자 중 하나와 일치하는지를 나타냅니다.
  -  EX) "[abc]"는 "a", "b", "c" 중 어느 문자와도 일치합니다.
  - "apple", "babble", "class" 와도 일치하지만, "dog", "letter" 와는 일치하지 않습니다.
6. [^] (캐럿): 대괄호 안에 포함되지 않은 문자와 일치합니다. NOT의 의미로 생각하면 될것같다. 대괄호안에 작성한문제를 제외한 문자와 일치하는지 나타냅니다. 
  - EX) "[abc]"는 "a", "b", "c" 중 어느 문자와도 일치하지 않습니다.
  - "apple", "babble", "class" 와도 일치하지않지만, "dog", "letter" 와는 일치합니다..
  - 추가내용) replaceAll 메서드에서 사용시에 "^" 의 의미는 문자열의 시작을 의미합니다. 반대로 "$"는 문자열의 끝을 의미합니다. 
7. \| (파이프): OR 조건을 나타냅니다.
8. \ (백슬래시): 다음에 오는 문자가 특수문자가 아님을 나타냅니다.

기본적인 특수문자 제거(대체) 및 정규 표현식을 사용한 제거(대체)하는 예시를 들어보겠습니다.

> (1) '.', '*', '+', '?', '|' (마침표, 별표, 더하기, 물음표, 파이프) 제거

/ (슬래시) 두개 또는 []안에 제거하고 싶은 문자를 작성하여 주면된다.

```java
String str = "He.llo.";
String replacedStr = str.replaceAll("\\.", "");
String replacedStr2 = str.replaceAll("[.]", "");

System.out.println(replacedStr);  // Hello
System.out.println(replacedStr2);  // Hello

String str2 = "a*b";
String replacedStr3 = str2.replaceAll("\\*", "");
String replacedStr4 = str2.replaceAll("[*]", "");

System.out.println(replacedStr3);  // ab
System.out.println(replacedStr4);  // ab 

String str3 = "a+b";
String replacedStr5 = str3.replaceAll("\\+", "");
String replacedStr6 = str3.replaceAll("[+]", "");

System.out.println(replacedStr5);  // ab
System.out.println(replacedStr6);  // ab

String str4 = "a?b";
String replacedStr7 = str4.replaceAll("\\?", "");
String replacedStr8 = str4.replaceAll("[?]", "");

System.out.println(replacedStr7);  // ab
System.out.println(replacedStr8);  // ab

String str5 = "apple|orange";
String replacedStr9 = str5.replaceAll("\\|", "");
String replacedStr10 = str5.replaceAll("[|]", "");

System.out.println(replacedStr9);  // appleorange
System.out.println(replacedStr10);  // appleorange
```

> (2) '[]', '()' (대괄호, 괄호)를 제거 

```java
String str = "[abc]";
String str2 = "(abc)";

String replacedStr = str.replaceAll("\\[|\\]", "");
String replacedStr = str2.replaceAll("\\(|\\)", "");
  
System.out.println(replacedStr);  // abc
System.out.println(replacedStr2);  // abc
```

(1) 과 동일하게 슬래시('/') 2개로 가능하며 '|' 는 '또는' 이라는 의미로 좌괄호 또는 우괄호를 제거해준다.

> (3) 대괄호를 사용한 문자 제거

`연속된 마침표 하나로 치환`

```java
String str = "asjgw....snbnw..snw...";

String replacedStr = str.replaceAll("[.]+", ".");
System.out.println(replacedStr);    // asjgw.snbnw.snw.
```

[]내부에 마침표(.)를 입력하여 마침표하나를 의미해주고 + 의 기호를 사용하여 앞에문자가 1회이상 반복되는 패턴을 읽어옵니다.

그 패턴 값을 replaceAll 메서드를 통하여 마침표(.) 하나로 대체하여 연속되는 마침표를 하나의 마침표로 치환합니다.

`조건에 따른 양끝의 문자열 삭제(치환)`

```java
String str = "asjgw.snbnw.snw.";

String replacedStr = str.replaceAll("^[.]|[.]$", "");
System.out.println(replacedStr);    // asjgw.snbnw.snw
```

대괄호 밖에서 사용된 ^,$는 각각 문자열의 시작과 끝을 의미하고 파이프(\|)의 의미는 또는 이라는 의미이다.

즉, 처음으로 시작하는 마침표 또는 마지막에 있는 마침표가 있으면 빈칸으로 대체하라는 코드이다.

`허용하지 않는 문자 제거`

```java
String str = "a123111sGjgw.sn*룸bnw.AAFJNsnw.";

String replacedStr = str.replaceAll("[^a-z]", "");
System.out.println(replacedStr);    // asjgwsnbnwsnw
```

replaceAll 내 코드의 의미는 a부터 z까지(a-z)의 문자가 아닌(^)문자는 전부 빈칸으로 대체하라는 의미이다. 

a-z라고 작성했으니 영어의 대문자 또한 포함하지 않으며, 소문자만 남고 전부 삭제된다. 

```java
String str = "a123111sGjgw.sn*룸bnw.AAFJNsnw.";

String replacedStr = str.replaceAll("[^0-9]", "");
System.out.println(replacedStr);    // 123111
```

이 코드는 숫자를 제외하고 전부 빈칸으로 대체하라는 의미이다.

```java
String str = "a123111sGjgw.sn*룸bnw.AAFJNsnw.";

String replacedStr = str.replaceAll("[^a-z0-9-_.]", "");
System.out.println(replacedStr);    // a123111sjgw.snbnw.snw.
```

이 코드는 소문자, 숫자, 하이픈, 언더바, 마침표를 제외하고 전부 빈칸으로 대체하라는 코드이다.

대괄호안에 코드를 어떻게 변경하는가에 따라서 원하는 데이터만 뽑아낼 수 있다.




***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}