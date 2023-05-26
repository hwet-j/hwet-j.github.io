---
title:  "[Java] 오버라이딩과 오버로딩"  
excerpt: "두 개의 차이점"

categories: # 분류하고싶은 카테고리 입력
  - Java
tags:     # 중요 키워드
  - [Term]

toc: true
toc_sticky: true

author: Hwet

date: 2023-05-26
last_modified_at: 2023-05-26
---

인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1](http://www.tcpschool.com/java/java_usingMethod_overloading)
[참고2](http://www.tcpschool.com/java/java_inheritance_overriding)
{: .notice--warning}

## 오버라이딩

오버라이딩은 상속관계 또는 포함관계에서 물려받은 메서드를 재정의 하는 것을 말한다.

자바에서 상속 시 private 멤버를 제외한 모든 메서드를 상속받는다. 이때, 상속 받은 메서드는 그대로 사용해도 되고 필요에 의해 재정의 가능하다. 

포함관계는 물려받은 모든 메서드를 재정의 해서 사용해야 한다. 

상속은 Is a 관계 포함은 Has a 관계를 말하며 추후에 다루도록 하겠다.

### 오버라이딩 사용법

1. 오버라이딩 하려는 메서드는 부모클래스의 메서드와 이름, 매개변수, 반환타입이 동일히야한다.
2. 접근제어자의 경우는 상위 클래스보다 좁은 범위로 변경이 가능하다. (부모에서 public이었지만, 자식에서 private 가능)
3. 상위 클래스 메서드보다 많은 수의 예외를 선언할 수 있다.

### 오버라이딩 예시

```java
class Woman{ //부모클래스
	public String name;
	public int age;
	//info 메서드
	public void info(){
		System.out.println("여자의 이름은 "+name+", 나이는 "+age+"살입니다.");
	}
}
class Job extends Woman{ 		//Woman클래스(부모클래스)를 상속받음 : 
	String job;
	public void info() {	//부모(Woman)클래스에 있는 info()메서드를 재정의
		super.info();
		System.out.println("여자의 직업은 "+job+"입니다.");
	}
}
public class OverTest {
	public static void main(String[] args) {
		//Job 객체 생성
		Job job = new Job();
		//변수 설정
		job.name = "유리";
		job.age = 30;
		job.job = "프로그래머";
		//호출
		job.info();
	}
}
```

> 부모클래스(Woman)에서 정의된 info()가 아닌 자식클래스(Job)에서 재정의된 메서드가 사용된다.
> 결과 : 여자의 직업은 프로그래머 입니다. 

## 오버로딩 

오버로딩은 같은 이름의 메서드를 정의하고, 매개변수의 타입, 개수, 순서를 다르게 하여 다양한 호출에 응답할 수 있게 된다.

메서드의 리턴타입은 중요하지 않다. 

### 오버로딩 사용법 

1. 메서드의 이름이 동일 해야한다. 
2. 매개변수의 타입 또는 순서 또는 개수가 달라야한다.


### 오버로딩 예시
```java
class OverloadingExample{
	void cat(){
		System.out.println("매개변수 없음");
	}
	void cat(int a){  // 매개변수 1개 타입 int
		System.out.printf("현재 %d마리의 고양이가 있습니다.\n", %d);
	}
	void cat(String b){ // 매개변수 1개 타입 String
		System.out.printf("고양이가 있는 곳은 %s입니다.\n", %s);
	}
}
```

> main메소드에서 호출 가정 한다면
  >	```
  > OverloadingExample ole = new OverloadingExample();
  >	ole.cat(); 		// => 매개변수 없음 출력
  >	ole.cat(4);		// => 현재 4마리의 고양이가 있습니다.
  >	ole.cat("동물병원"); // => 고양이가 있는 곳은 동물병원입니다.
  > ```

***

`오버라이딩, 오버로딩 같은 경우에는 객체 지향 프로그래밍의 특징 중 하나인 다형성을 구현하는 방식으로 매우 중요하다`

***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}