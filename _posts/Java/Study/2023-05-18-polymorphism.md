---
title:  "[Java] 다형성 - 형변환, 오버라이딩, 오버로딩"
excerpt: "다형성을 알아보자"

categories:
  - Java
tags:
  - [Term]

toc: true
toc_sticky: true

author: Hwet

date: 2023-05-18
last_modified_at: 2023-10-05
---


인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1](https://min-nine.tistory.com/143)
[참고2](https://kadosholy.tistory.com/99)
{: .notice--warning}


### 다형성

#### 형변환
형변환은 상속받은 객체에 대해서 멤버들에 대한 사용범위가 달라진다는 것을 의미한다.

형변환은 강제형변환, 자동형변환이 존재하는데 이후 형변환을 따로 자세히 포스팅할 예정이다.

- 업캐스팅 (upcasting) : 자식클래스의 타입의 레퍼런스 변수(자식클래스의 인스턴스)를 부모클래스 타입으로 형변환 하는것.
  - 업캐스팅은 경우 자동형변환이 가능하여 명시하지 않아도된다
  - 부모 클래스로 캐스팅된다는 것은 멤버의 개수 감소를 의미한다. (자식 클래스에만 존재하는 속성과 메서드는 사용할 수 없다)
  - 자식클래스에서 오버라이딩된 메서드가 존재할 경우 업캐스팅 하여도 자식클래스에서 오버라이딩한 메서드가 실행된다.
- 다운캐스팅 (downcasting) : 부모클래스 타입의 레퍼런스 변수(자식클래스의 인스턴스)를 자식 클래스 타입으로 형변환 하는 것.
  - 다운 캐스팅은 반드시 형변환 타입을 명시해야한다.
  - 업캐스팅한 객체를 다시 자식클래스로 되돌려 복구하는 목적을 둔다.(기능과 속성 회복)

> - 상속 관계에 있는 클래스 간에는 자동형변환, 강제형변환이 가능하다. (다형성)
> - 전혀 다른 두 클래스 간에 타입 변환은 불가능할 뿐더러 instanceof 키워드사용도 불가 하다.

 
<br>
<strong style="color:red">주의 사항!</strong>

> - 인스턴스와 타입을 구분해야한다. (인스턴스 자체가 타입을 포함하지 않는 객체라면 다운캐스팅 불가)
> - 업캐스팅의 경우 타입으로 변환하면 하면 되기 때문에 명시하지 않아도 되지만, 다운캐스팅의 경우 가리키는 참조변수가 명확하지 않기 때문에 명시해야하는것이다.
> - 참조 캐스팅을 잘못해도 환경에 따라 컴파일 에러가 발생하지 않기도 하고 런타임 에러가 발생하는 경우가 있을 수 있다. 익숙하지 않다면 디버깅을 통해 파악해 보는것이 좋다.
> - insteadof 연산자를 사용하여 참조타입을 포함하는지 true/false로 반환해주는데, 참조형 타입에만 사용할 수 있으며 아예 연관없는 참조 타입을 확인하려고 하면 컴파일 에러가 발생하기도 한다.<br>

#### 오버로딩(Overloading)
오버로딩은 같은 이름의 함수(메서드)를 여러개 정의하고, <strong style="color:red">매개변수의 타입과 개수, 순서를 다르게</strong> 하여 
다양한 호출에 응답할 수 있게 된다. <br> --> 메서드의 리턴타입은 중요하지 않다. 

> - 메서드명은 동일하지만 실질적인 형상은 달라 다른 기능을 한다. 
> - 생성자도 메서드의 일종으로 볼 수 있으며, 생성자 또한 오버로딩 가능하다.

```java
class OverloadingExample{
	void cat(){
		System.out.println("매개변수 없음");
	}
	void cat(int a){
		// System.out.println("매개변수 하나 int 타입");
		System.out.printf("현재 %d마리의 고양이가 있습니다.\n", %d);
		// > 동일한 메소드지만 
	}
	void cat(String b){
		//System.out.println("매개변수 하나 String타입");
		System.out.printf("고양이가 있는 곳은 %s입니다.\n", %s);
	}
}
	// => main메소드에서 호출 가정 한다면
	OverloadingExample ole = new OverloadingExample(); // 
	ole.cat(); 		// => 매개변수 없음 출력
	ole.cat(4);		// => 현재 4마리의 고양이가 있습니다. 출력
	ole.cat("동물병원"); // => 고양이가 있는 곳은 동물병원입니다. 출력
```

#### 오버라이딩(Overriding)
상위 클래스가 가지고 있는 맴버변수가 하위 클래스로 상속되는 것처럼 상위 클래스가 가지고 있는 메서드도 하위 클래스로
상속되어 하위 클래스에 사용할 수 있습니다. 또한, 하위클래스에서 메서드를 재정의해서 사용가능하다. <br>

<strong style="color:red">메서드의 이름, 매개변수, 반환형이 모두 동일한 상황에서 상속받은 메서드를 덮어씌운것</strong>이라고 생각하면 된다. <br>

> - 생성자 호출 시 해당 생성자의 오버라이딩 된 메서드중 가장 하위(자식) 클래스의 메서드가 사용된다.
> - 부모클래스의 메서드를 재정의한 메서드로 부모와 생김새는 유사하지만 다른 기능을 수행한다. (다형성)

```java
class Parent{ //부모클래스
	public String name;
	public int age;
    
	public void info(){
		System.out.println("부모클래스의 메서드");
	}
}

class Child extends Parent{ 		// 자식클래스 : 부모클래스를 상속받음 
	String job;
    
	public void info() {	//부모클래스에 있는 info()메서드를 재정의
		System.out.println("자식클래스에서 오버라이딩된 메서드");
	}
}

public class OverTest {
	public static void main(String[] args) {
    // 객체 생성으로 실험
    // Parent 타입의 참조변수를 Parent로 생성자 초기화
		Parent Test1 = new Parent();
    Test1.info();       // 타입도 Parent 초기화도 Parent이므로 당연히 결과는 "부모클래스의 메서드"
        
    // Parent 타입의 참조변수를 Child로 생성자 초기화
		Parent Test2 = new Child();
    Test2.info();       // 타입은 Parent이지만 초기화가 Child로 되었으므로 결과는 "자식클래스에서 오버라이딩된 메서드"
    
    // Child 타입의 참조변수를 Child로 생성자 초기화
    Child Test3 = new Child();
    Test3.info();       // 타입도 Child 초기화도 Child이므로 결과는 "자식클래스에서 오버라이딩된 메서드"
    
    // 형변환을 하여도 변하지 않음 (강제형변환을 말하는것)
    Parent Test4 = (Parent) Test3;
    Test4.info();       // Child로 초기화된 참조변수를 Parent로 강제형변환하였지만, 결과는 "자식클래스에서 오버라이딩된 메서드"
    
	}
}
// ===> 여자의 직업은 프로그래머입니다. 출력
```

상속, 인터페이스, 추상클래스에 대한 개념을 추가로 공부하면 이해하는데 도움이 될거같다.



***
<br>

    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}