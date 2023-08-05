---
title:  "[Web/JSP] JSTL과 EL?"  
excerpt: "JSP Standard Tag Library"

categories: # 분류하고싶은 카테고리 입력
  - Web
tags:     # 중요 키워드
  - [HTML, JSP]

toc: true
toc_sticky: true
published: true

author: Hwet

date: 2023-07-21
last_modified_at: 2023-07-27
---

인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.    
[참고1](https://usefultoknow.tistory.com/entry/JSTLJSP-Standard-Tag-Library%EC%9D%B4%EB%9E%80)
{: .notice--warning}


## JSTL이란?

JSPL(JSP Standard Tag Library)란 자바코드를 캡슐화하는 기술을 자신만의 태그로 만드는 것을 말한다.

&lt;jsp:include&gt; 나 &lt;jsp:usebean&gt; 과 같은 것을 말하며 연산이나 조건문,반복문을 통해 DB를 편하게 처리할 수 있게한다.

### 태그의 종류

#### 1. Core (prefix : c)

URI : [http://java.sun.com/jsp/jstl/core]

주요기능 : 실행 흐름을 제어하는 기능. 변수 처리, 조건문, 반복문 등 

사용방법 : `<c:태그명>`형태로 선언하여 사용한다. (태그명 : set, if, choose, forEach....)

#### 2. Formatting (prefix : fmt)

URI : [http://java.sun.com/jsp/jstl/fmt]

주요기능 : 날짜, 숫자, 메시지 등의 포멧 형식을 다양하게 제공해 텍스트의 형식을 변환하는 기능 / 국제화, 다국어 기능 제공

사용방법 : `<fmt:태그명>`형태로 선언하여 사용한다. (태그명 : formatNumber, formatDate, message....)

#### 3. DataBase (prefix : sql)

URI : [http://java.sun.com/jsp/jstl/sql]

주요기능 : DataBase와의 상호작용을 지원한다. 입력, 수정, 삭제, 조회 등의 기능을 제공

사용방법 : `<sql:태그명>`형태로 선언하여 사용한다. (태그명 : setDataSource, query, update....)

#### 4. XML (prefix : x)

URI : [http://java.sun.com/jsp/jstl/xml]

주요기능 : XML 모듈은 XML 문서를 처리하는데 사용됩니다. XML 데이터의 파싱, 생성, 변환 등의 작업을 수행

사용방법 : `<x:태그명>`형태로 선언하여 사용한다. (태그명 : parse, out, transform....)

#### 5. Function (prefix : fn)

URI : [http://java.sun.com/jsp/jstl/functions]

주요기능 : 자주 사용되는 함수를 제공하여 문자열 처리, 수학적 연산, 컬렉션 처리 등을 간편하게 수행

사용방법 : `<fn:태그명>`형태로 선언하여 사용한다. (태그명 : concat(), substring(), contains(), toUpperCase()....)


### JSTL 특징

JSTL 기능 확장: JSTL은 기본적인 기능을 제공하지만, 확장하여 사용자 정의 태그를 만들 수도 있습니다. 이를 통해 사용자가 필요에 맞게 태그를 정의하고 재사용할 수 있습니다. 
커스텀 태그 라이브러리를 만들어 활용할 수 있습니다.

JSTL의 장점: JSTL을 사용하면 JSP 페이지의 가독성을 높일 수 있고, 자바 코드의 양을 줄여 유지보수를 용이하게 합니다. 
또한 일반적으로 사용되는 기능을 태그로 제공하므로 개발 시간을 단축할 수 있습니다.

EL(Expression Language): JSTL은 EL(Expression Language)을 지원합니다. 
EL은 JSP 페이지에서 Java 코드 없이도 변수의 값을 출력하거나 연산을 수행하는 간단한 표현식을 사용할 수 있게 해줍니다.

권장 사용법: JSTL은 주로 뷰(View)에서 사용되며, 비즈니스 로직은 서블릿이나 Java 클래스에서 처리하는 것이 좋습니다. 
이렇게 구분하여 MVC 패턴을 따르면 코드의 유지보수성이 더욱 향상됩니다.






*** 



***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}