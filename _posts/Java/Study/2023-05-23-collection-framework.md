---
title:  "[Java] 컬렉션 프레임워크"  
excerpt: "컬렉션 프레임워크는 배열과 어떻게 다를까?"

categories: # 분류하고싶은 카테고리 입력
  - Java
tags:     # 중요 키워드
  - [Term]

toc: true
toc_sticky: true

author: Hwet

date: 2023-05-23
last_modified_at: 2023-05-23
---

인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1](http://www.tcpschool.com/java/java_collectionFramework_concept)
{: .notice--warning}

## 배열

### 배열의 선언

```java
// 배열 크기 할당 X, 초기화 X, 배열의 참조변수만 선언 (두가지는 동일한 의미, 아무거나 사용)
int[] arr;
int arr;

// 선언과 배열 크기할당 (초기화 X)
int[] arr = new int[10]; // --> int형 배열의 default값은 0이다. (크기가 10이며 초기값이 0인 배열 생성됨)		
String[] strArr = new String[10]; // --> 문자열 배열의 default값은 null이다. (크기가 10이며 초기값이 null인 배열 생성됨)		

// 배열의 선언, 크기설정 따로 지정
int[] arr;
arr = new int[10]; // 크기가 10이며 초기값이 0인 배열 생성됨

// 선언과 동시에 초기값 지정(배열의 크기도 설정됨)
int[] arr = {1,2,3,4,5};
int[] arr = new int[] {1,2,3,4,5}; // [] 을 잊지않도록 하자.
String[] number = {"하나", "둘", "셋", "넷", "다섯"};

// 동일한 방식으로 2차원 배열 선언이 가능하다.
int[][] arr = new int[4][9]; // 4의 크기를 가진 배열을 9개 지닌 2차원 배열 선언 (마찬가지로 초기값은 0으로 설정됨)
```

<p><details>
<summary style="color:red;">기본타입과 참조타입</summary><!-- summary 아래 한칸 공백 필요 -->

> 기본 타입 : byte, char, short, int, long, float <br>
> 참조 타입 : 배열, 열거, 클래스, 인터페이스 <br>
> ==> 기본타입은 실제값을 변수에 저장하지만 참조타입은 메모리의 주소값을 변수 안에 저장한다.
</details></p>


### 배열의 출력

배열은 참조변수 이므로 배열명을 기입하여 출력하려하면 주소값을 반환한다.

그렇기 때문에 다른 방식을 사용하여 배열을 출력해야한다.

```java
int[] arr = {1,2,3,4,5};
// 1. 기본 for문 사용     
for(int i = 0; i < arr.length; i++) {
       System.out.println(arr[i]);
}
// 2. 향상된 for문 사용 ( 배열이 for문의 변수로 사용 )
for(int number : arr) {
       System.out.println(number);
}
// 3. Arrays 클래스의 toString() 메소드 사용
System.out.println(Arrays.toString(arr));

```

### 배열의 복사

- 얕은 복사(Shallow Copy) : 복사된 배열이나 원본배열이 변경될 때 서로 간의 값이 같이 변경됩니다. (주소값을 복사/공유)
- 깊은 복사(Deep Copy) : 복사된 배열이나 원본배열이 변경될 때 서로 간의 값이 바뀌지 않습니다. (주소에 저장된 요소를 저장)

```java
public class Array_Copy{
    public static void main(String[] args)  {
        int[] a = { 1, 2, 3, 4 };
        int[] b = new int[a.length]; 
        for (int i = 0; i < a.length; i++) {
            b[i] = a[i];
        }
    }
}
// ==> for문을 사용하여 각 요소를 복사하고 있으므로 깊은 복사이다.

public class Array_Copy{
    public static void main(String[] args)  {
        int[] a = { 1, 2, 3, 4 };
        int[] b = a;
    }
}
// ==> a라는 변수명에는 주소값이 저장되어있는데 b는 a를 대입하게 되어 b에 a의 주소값이 저장된다. 그러므로 주소값을 공유하게 되고 이는 얕은 복사이다.

```

### 배열을 복사하는 여러가지 메서드

#### <1> Object.clone() : 가장 보편적으로 사용되며 쉽게 배열을 깊은 복사할 수 있다.

```java
public class Array_Copy{
    public static void main(String[] args)  {
        int[] a = { 1, 2, 3, 4 };
        int[] b = a.clone();
    }
}
```

#### <2> Arrays.copyOf() : 배열의 시작점 ~ 원하는 length까지 배열의 깊은 복사를 할 수 있다. 

```java
import java.util.Arrays;

public class Array_Copy{
    public static void main(String[] args)  {
        int[] a = { 1, 2, 3, 4 };
        int[] b = Arrays.copyOf(a, a.length);
    }
}
```

#### <3> Arrays.copyOfRange() : 원하는시작점 ~ 원하는 length까지 깊은 복사를 할 수 있다. 

```java
import java.util.Arrays;

public class Array_Copy{
    public static void main(String[] args)  {
        int[] a = { 1, 2, 3, 4 };
        int[] b = Arrays.copyOfRange(a, 1, 3);
    }
}
```

#### <4> System.arraycopy() : 지정된 배열을 대상 배열의 지정된 위치에 복사합니다.

```java
public class Array_Copy{
    public static void main(String[] args)  {
        int[] a = { 1, 2, 3, 4 };
        int[] b = new int[a.length];
        System.arraycopy(a, 0, b, 0, a.length);
    }
}
```




***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}