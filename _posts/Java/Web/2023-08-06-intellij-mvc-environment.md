---
title:  "[Web/IDE] IntelliJ Community 환경에서 MVC환경 구축"  
excerpt: "인텔리제이 커뮤니티 환경에서 MVC환경 구축하기"

categories: # 분류하고싶은 카테고리 입력
  - Web
tags:     # 중요 키워드
  - [IntelliJ, Spring]

toc: true
toc_sticky: true
published: true

author: Hwet

date: 2023-08-06
last_modified_at: 2023-08-06
---

인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.  
[참고](https://www.youtube.com/watch?v=tWgBlpgk0ls)
{: .notice--warning}


## IntelliJ에서 MVC web 환경 프로젝트 구축하기

2023년도 이전에는 '프레임워크 기능 추가' 를 클릭하여 편리하게 Spring MVC 환경을 구축할 수 있었지만, 2023년도에 업데이트 하면서 해당 기능으로 환경을 구축할 수 없게 되었다.

인터넷에 검색하면 대부분 이 방법으로 설명을 진행하고 있기 때문에 이 업데이트된 상황에서 구축하는 방법을 작성하고자 한다.

### 📌 프로젝트 생성 

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/0a1ae576-3910-47f8-8a7c-3983643fd0a5){:width="50%"}

이름 : `프로젝트 명`

JDK : `11이상의 버전`

Archetype : `org.apache.maven.archetypes:maven-archetype-webapp`

위의 설정값대로 설정한 이후 '생성'을 클릭하여 프로젝트를 생성한다. 생성이 되면 해당 프로젝트에 필요한 정보를 다운받는다. 

### 📌 프로젝트 구조 확인

> .idea : 프로젝트 관리에 사용되는 정보가 들어있는 폴더(설정 정보, 메타데이터)
 
- workspace.xml: 인텔리제이의 전역 설정 정보를 담고 있습니다.
- misc.xml: 기타 설정 정보를 담고 있습니다.
- compiler.xml: 컴파일 옵션, 경로 설정, 어노테이션 프로세서 등과 컴파일 관련 설정 정보를 포함합니다. 이 파일을 통해 프로젝트가 어떻게 컴파일되는지에 대한 세부 사항을 조정할 수 있습니다.
- encoding.xml: 프로젝트의 소스 코드 및 리소스 파일의 문자 인코딩 설정을 담고 있습니다. 프로젝트 내의 파일들이 어떤 문자 인코딩으로 저장되고 읽혀져야 하는지에 대한 정보를 포함합니다.
- uiDesigner.xml: GUI 디자인 도구인 UI Designer의 설정 정보를 담고 있습니다. UI Designer는 사용자 인터페이스를 시각적으로 디자인할 수 있는 도구로, Java Swing 애플리케이션의 GUI를 만드는 데 도움을 줍니다.
- jarRepositories.xml: 라이브러리 관리 및 의존성 설정과 관련된 정보를 담고 있습니다. 프로젝트에서 사용하는 외부 라이브러리나 의존성을 어떻게 가져와야 하는지에 대한 정보를 저장합니다.

`해당 폴더는 프로젝트 설정 관련하여 중요한 정보를 포함하므로 함부로 삭제하거나 수정해서는 안되며, 협업을 하거나 다른 환경에서 프로젝트를 사용하기 위해서는 해당 폴더도 유지해야 올바른 동작을 보장할 수 있습니다.`

> src : 프로젝트의 소스 코드와 리소스 파일이 저장되는 위치

- src/main/java : 프로젝트의 java 소스 코드 파일들이 저장되며, 실제 기능과 로직을 작성합니다.
- src/main/resources : 리소스 파일들이 저장되며, 리소스 파일은 코드에 포함되지 않고 별도로 관리됩니다. (설정, 프로퍼티, XML, 이미지 등 ...)
- src/main/webapp : 주로 웹 어플리케이션의 웹 자원(이미지, CSS, JavaScript 등)들을 포함하는 곳

> pom.xml : Maven 프로젝트의 핵심 설정 파일 / 프로젝트의 의존성, 빌드 설정, 프러그인 설정 등을 정의한다.

- 프로젝트의 이름, 버전, 그룹 ID, 아티팩트 ID 등의 기본 정보
- 프로젝트의 의존성: 프로젝트가 사용하는 외부 라이브러리 및 다른 모듈과의 의존성을 정의
- 빌드 설정 : 빌드 시에 어떤 작업을 수행할지, 어떤 플러그인을 사용할지 등을 정의
- 리포지토리 정보 : 프로젝트의 라이브러리를 저장하거나 다운로드할 리포지토리 정보
- 테스트 및 리소스 디렉토리 설정
- 라이선스 정보

등을 포함할 수 있다. 만약 아직 폴더 또는 파일이 존재하지 않더라도 이후에 생성 및 수정할 예정이다. 

### 📌 프로젝트 구조 변경(설정)

> pom.xml

```xml
  <!-- 프로젝트의 전역 속성 설정 -->
  <properties>
    <!-- 프로젝트의 소스 코드 인코딩을 UTF-8로 설정 -->
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <!-- 컴파일 시에 사용할 Java 버전 설정 -->
    <maven.compiler.source>11</maven.compiler.source>
    <!-- 컴파일한 클래스 파일의 대상 Java 버전 설정 -->
    <maven.compiler.target>11</maven.compiler.target>
    <!-- 프로젝트에서 사용할 Java 버전 설정 -->
    <java-version>11</java-version>
    <!-- Spring Framework 버전 설정 -->
    <org.springframework-version>5.3.20</org.springframework-version>
    <!-- SLF4J (Simple Logging Facade for Java) 버전 설정 -->
    <org.slf4j-version>1.7.25</org.slf4j-version>
  </properties>
  <dependencies>

    <!-- Spring 관련 의존성 설정 -->
    <dependency>
      <!-- 의존성의 그룹 ID 설정: org.springframework -->
      <groupId>org.springframework</groupId>
      <!-- 의존성의 아티팩트 ID 설정: spring-context -->
      <artifactId>spring-context</artifactId>
      <!-- 의존성의 버전 설정: 위에서 정의한 org.springframework-version 값 사용 -->
      <version>${org.springframework-version}</version>
      <!-- 의존성 제외 설정: Commons Logging 대신 SLF4j 사용 -->
      <exclusions>
        <!-- 의존성 제외: groupId가 commons-logging, artifactId가 commons-logging인 경우 -->
        <exclusion>
          <groupId>commons-logging</groupId>
          <artifactId>commons-logging</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
    <!-- Spring Web MVC 관련 의존성 설정 -->
    <dependency>
      <!-- 의존성의 그룹 ID 설정: org.springframework -->
      <groupId>org.springframework</groupId>
      <!-- 의존성의 아티팩트 ID 설정: spring-webmvc -->
      <artifactId>spring-webmvc</artifactId>
      <!-- 의존성의 버전 설정: 위에서 정의한 org.springframework-version 값 사용 -->
      <version>${org.springframework-version}</version>
    </dependency>

    <!-- Logging -->
    <!-- SLF4J (Simple Logging Facade for Java) 관련 의존성 설정 -->
    <dependency>
      <!-- 의존성의 그룹 ID 설정: org.slf4j -->
      <groupId>org.slf4j</groupId>
      <!-- 의존성의 아티팩트 ID 설정: slf4j-api -->
      <artifactId>slf4j-api</artifactId>
      <!-- 의존성의 버전 설정: 위에서 정의한 org.slf4j-version 값 사용 -->
      <version>${org.slf4j-version}</version>
    </dependency>

    <dependency>
      <!-- 의존성의 그룹 ID 설정: org.slf4j -->
      <groupId>org.slf4j</groupId>
      <!-- 의존성의 아티팩트 ID 설정: jcl-over-slf4j -->
      <artifactId>jcl-over-slf4j</artifactId>
      <!-- 의존성의 버전 설정: 위에서 정의한 org.slf4j-version 값 사용 -->
      <version>${org.slf4j-version}</version>
      <!-- 의존성의 스코프 설정: 런타임 스코프로 설정 -->
      <scope>runtime</scope>
    </dependency>

    <!-- @Inject -->
    <dependency>
      <!-- 의존성의 그룹 ID 설정: javax.inject -->
      <groupId>javax.inject</groupId>
      <!-- 의존성의 아티팩트 ID 설정: javax.inject -->
      <artifactId>javax.inject</artifactId>
      <!-- 의존성의 버전 설정: 1 -->
      <version>1</version>
    </dependency>

    <!-- javax.servlet-api 의존성 설정 -->
    <dependency>
      <!-- 의존성의 그룹 ID 설정: javax.servlet -->
      <groupId>javax.servlet</groupId>
      <!-- 의존성의 아티팩트 ID 설정: javax.servlet-api -->
      <artifactId>javax.servlet-api</artifactId>
      <!-- 의존성의 버전 설정: 3.1.0 -->
      <version>3.1.0</version>
      <!-- 의존성의 스코프 설정: provided -->
      <scope>provided</scope>
    </dependency>

    <!-- jsp-api 의존성 설정 -->
    <dependency>
      <!-- 의존성의 그룹 ID 설정: javax.servlet.jsp -->
      <groupId>javax.servlet.jsp</groupId>
      <!-- 의존성의 아티팩트 ID 설정: jsp-api -->
      <artifactId>jsp-api</artifactId>
      <!-- 의존성의 버전 설정: 2.1 -->
      <version>2.1</version>
      <!-- 의존성의 스코프 설정: provided -->
      <scope>provided</scope>
    </dependency>

    <!-- jstl 의존성 설정 -->
    <dependency>
      <!-- 의존성의 그룹 ID 설정: javax.servlet -->
      <groupId>javax.servlet</groupId>
      <!-- 의존성의 아티팩트 ID 설정: jstl -->
      <artifactId>jstl</artifactId>
      <!-- 의존성의 버전 설정: 1.2 -->
      <version>1.2</version>
    </dependency>

    <!-- DB -->
    <!-- MySQL JDBC 드라이버 의존성 설정 -->
    <dependency>
      <!-- 의존성의 그룹 ID 설정: mysql -->
      <groupId>mysql</groupId>

      <!-- 의존성의 아티팩트 ID 설정: mysql-connector-java -->
      <artifactId>mysql-connector-java</artifactId>

      <!-- 의존성의 버전 설정: 8.0.22 -->
      <version>8.0.22</version>
    </dependency>

    <!-- MyBatis 의존성 설정 -->
    <dependency>
      <!-- 의존성의 그룹 ID 설정: org.mybatis -->
      <groupId>org.mybatis</groupId>
      <!-- 의존성의 아티팩트 ID 설정: mybatis -->
      <artifactId>mybatis</artifactId>
      <!-- 의존성의 버전 설정: 3.5.6 -->
      <version>3.5.6</version>
    </dependency>

    <!-- MyBatis-Spring 의존성 설정 -->
    <dependency>
      <!-- 의존성의 그룹 ID 설정: org.mybatis -->
      <groupId>org.mybatis</groupId>
      <!-- 의존성의 아티팩트 ID 설정: mybatis-spring -->
      <artifactId>mybatis-spring</artifactId>
      <!-- 의존성의 버전 설정: 1.3.2 -->
      <version>1.3.2</version>
    </dependency>

    <!-- Spring JDBC 의존성 설정 -->
    <dependency>
      <!-- 의존성의 그룹 ID 설정: org.springframework -->
      <groupId>org.springframework</groupId>
      <!-- 의존성의 아티팩트 ID 설정: spring-jdbc -->
      <artifactId>spring-jdbc</artifactId>
      <!-- 의존성의 버전 설정: 위에서 정의한 org.springframework-version 값 사용 -->
      <version>${org.springframework-version}</version>
    </dependency>

    <!-- Tomcat DBCP (Database Connection Pool) 의존성 설정 -->
    <dependency>
      <!-- 의존성의 그룹 ID 설정: org.apache.tomcat -->
      <groupId>org.apache.tomcat</groupId>
      <!-- 의존성의 아티팩트 ID 설정: tomcat-dbcp -->
      <artifactId>tomcat-dbcp</artifactId>
      <!-- 의존성의 버전 설정: 9.0.31 -->
      <version>9.0.31</version>
    </dependency>


    <!-- commons-fileupload 의존성 설정 -->
    <dependency>
      <!-- 의존성의 그룹 ID 설정: commons-fileupload -->
      <groupId>commons-fileupload</groupId>
      <!-- 의존성의 아티팩트 ID 설정: commons-fileupload -->
      <artifactId>commons-fileupload</artifactId>
      <!-- 의존성의 버전 설정: 1.3.1 -->
      <version>1.3.1</version>
    </dependency>

    <!-- Lombok -->
    <dependency>
      <!-- 의존성의 그룹 ID 설정: org.projectlombok -->
      <groupId>org.projectlombok</groupId>
      <!-- 의존성의 아티팩트 ID 설정: lombok -->
      <artifactId>lombok</artifactId>
      <!-- 의존성의 버전 설정: 1.18.12 -->
      <version>1.18.12</version>
      <!-- 의존성의 스코프 설정: provided -->
      <scope>provided</scope>
    </dependency>

    <!-- json -->
    <!-- json-simple 의존성 설정 -->
    <dependency>
      <!-- 의존성의 그룹 ID 설정: com.googlecode.json-simple -->
      <groupId>com.googlecode.json-simple</groupId>
      <!-- 의존성의 아티팩트 ID 설정: json-simple -->
      <artifactId>json-simple</artifactId>
      <!-- 의존성의 버전 설정: 1.1.1 -->
      <version>1.1.1</version>
    </dependency>

    <!-- gson 의존성 설정 -->
    <dependency>
      <!-- 의존성의 그룹 ID 설정: com.google.code.gson -->
      <groupId>com.google.code.gson</groupId>
      <!-- 의존성의 아티팩트 ID 설정: gson -->
      <artifactId>gson</artifactId>
      <!-- 의존성의 버전 설정: 2.8.2 -->
      <version>2.8.2</version>
    </dependency>

    <!-- jackson-core -->
    <!-- jackson-core 의존성 설정 -->
    <dependency>
      <!-- 의존성의 그룹 ID 설정: com.fasterxml.jackson.core -->
      <groupId>com.fasterxml.jackson.core</groupId>
      <!-- 의존성의 아티팩트 ID 설정: jackson-core -->
      <artifactId>jackson-core</artifactId>
      <!-- 의존성의 버전 설정: 2.9.2 -->
      <version>2.9.2</version>
    </dependency>

    <!-- junit 의존성 설정 -->
    <dependency>
      <!-- 의존성의 그룹 ID 설정: junit -->
      <groupId>junit</groupId>
      <!-- 의존성의 아티팩트 ID 설정: junit -->
      <artifactId>junit</artifactId>
      <!-- 의존성의 버전 설정: 4.11 -->
      <version>4.11</version>
      <!-- 의존성의 스코프 설정: test -->
      <scope>test</scope>
    </dependency>

  </dependencies>

  <build>
    <!-- 최종 생성될 웹 애플리케이션의 이름 설정 -->
    <finalName>SpringMVC</finalName>

    <pluginManagement>
      <!-- Maven 플러그인 버전을 명시하여 기본 설정을 재정의 -->
      <plugins>
        <!-- maven-clean-plugin 설정 -->
        <plugin>
          <artifactId>maven-clean-plugin</artifactId>
          <version>3.1.0</version>
        </plugin>

        <!-- maven-resources-plugin 설정 (참고:http://maven.apache.org/ref/current/maven-core/default-bindings.html#Plugin_bindings_for_war_packaging) -->
        <plugin>
          <artifactId>maven-resources-plugin</artifactId>
          <version>3.0.2</version>
        </plugin>

        <!-- maven-compiler-plugin 설정 -->
        <plugin>
          <artifactId>maven-compiler-plugin</artifactId>
          <version>3.8.0</version>
        </plugin>

        <!-- maven-surefire-plugin 설정 -->
        <plugin>
          <artifactId>maven-surefire-plugin</artifactId>
          <version>2.22.1</version>
        </plugin>

        <!-- maven-war-plugin 설정 -->
        <plugin>
          <artifactId>maven-war-plugin</artifactId>
          <version>3.2.2</version>
        </plugin>

        <!-- maven-install-plugin 설정 -->
        <plugin>
          <artifactId>maven-install-plugin</artifactId>
          <version>2.5.2</version>
        </plugin>

        <!-- maven-deploy-plugin 설정 -->
        <plugin>
          <artifactId>maven-deploy-plugin</artifactId>
          <version>2.8.2</version>
        </plugin>
      </plugins>
    </pluginManagement>

    <!-- 프로젝트에 직접 적용되는 플러그인 설정 -->
    <plugins>
      <!-- maven-compiler-plugin 설정 -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
          <!-- 컴파일 소스 및 대상 버전 설정 -->
          <source>11</source>
          <target>11</target>
        </configuration>
      </plugin>
    </plugins>
  </build>
```

pom.xml의 project 태그 내에 해당 정보를 복사해서 넣어주면, 해당 파일 에디터 우측 상단에 Maven 변경 내역 로드 버튼을 클릭하여 설정된 정보를 다운로드 할 수 있다.

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/5c73dad5-b1a4-47c5-b12d-b8bcc770c9c3){:width="50%"}

> src/main/webapp/WEB-INF 폴더 설정

1. web.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1">

  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>
      /WEB-INF/root-context.xml
    </param-value>
  </context-param>

  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>

  <servlet>
    <servlet-name>appServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>/WEB-INF/servlet-context.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
  </servlet>

  <servlet-mapping>
    <servlet-name>appServlet</servlet-name>
    <url-pattern>/</url-pattern>
  </servlet-mapping>

  <filter>
    <filter-name>encodingFilter</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    <init-param>
      <param-name>encoding</param-name>
      <param-value>UTF-8</param-value>
    </init-param>
  </filter>

  <filter-mapping>
    <filter-name>encodingFilter</filter-name>
    <url-pattern>/*</url-pattern>
  </filter-mapping>

</web-app>
```

2. servlet-context.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans:beans xmlns="http://www.springframework.org/schema/mvc"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xmlns:beans="http://www.springframework.org/schema/beans"
             xmlns:context="http://www.springframework.org/schema/context"
             xsi:schemaLocation="http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc.xsd
		http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

    <annotation-driven />

    <!-- Handles HTTP GET requests for /resources/** by efficiently serving up static resources in the ${webappRoot}/resources directory -->
    <resources mapping="/resources/**" location="/resources/"/>

    <!-- Resolves views selected for rendering by @Controllers to .jsp resources in the /WEB-INF/views directory -->
    <beans:bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <beans:property name="prefix" value="/views/" />
        <beans:property name="suffix" value=".jsp" />
    </beans:bean>

    <!-- 프로젝트 패키지이름 확인 -->
    <context:component-scan base-package="com.codingrecipe.project01" />

</beans:beans>
```

context 태그의 설정에서 기본 패키지를 설정가능하다. 이후 패키지 생성 시에 여기에 설정된 값을 사용해야한다.

beans 태그의 속성명 prefix의 value값 설정으로 views 파일의 경로 설정이 가능하다.

3. root-context.xml

```xml 
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:jdbc="http://www.springframework.org/schema/jdbc"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/jdbc http://www.springframework.org/schema/jdbc/spring-jdbc.xsd
		http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd
		http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd">


</beans>
```

3개의 파일을 해당 폴더 내에 만들어주면 된다. 

> src/main

main 폴더에서 우클릭하여 '새로만들기 - 경로'를 클릭하면 하단에 Maven 소스 디렉토리에 java, resource가 있는데 클릭하여 생성해 준다. 

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/17ad7253-8b8a-4a14-ad4e-a971524b0e85){:width="50%"}

> src/main/java

servlet-context.xml 파일에서 설정하였던 프로젝트 패키지 경로대로 패키지를 생성한다. 

생성된 패키지 내에 'controller'패키지 생성, 'HomeController.java'파일 생성하여 서버 실행시 실행될 기본 루트를 설정한다.

```java
package com.codingrecipe.project01.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class HomeController {
    @GetMapping("/")
    public String index(){
        return "index";
    }
}
```

클래스명은 기본페이지의 정보를 표시하는 정보임을 알 수 있도록 설정하고 필자는 참고한 유투브에서 설정한 값을 그대로 사용하였습니다.

@Controller : 이 어노테이션으로 해당 클래스가 컨트롤러 역할을 수행한다는 것을 나타낸다.

@GetMapping("/") : Http GET 요청을 처리하는 경로를 처리하며, "/"의 경로로 들어오는 요청을 이 메서드가 처리한다.

return "index" : index라는 이름의 뷰를 찾아 클라이언트에게 보여줍니다.

> src/main/webapp/views ---> servlet-context.xml에서 설정한 경로에 작성(각자 설정한 경로)

```JSP
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Insert title here</title>
</head>
<body>
    <h2>MVC 패턴 구현 테스트중...</h2>
</body>
</html>
```

이전에 HomeController에 작성한 정보를 토대로 index.jsp 파일을 생성한다. (return값에 설정한 String값을 파일명으로 설정)

인텔리제이 Community 버전은 jsp파일의 자동생성 기능을 지원하지 않기 때문에 기본적인 형식을 따로 저장해두고 사용하는 것을 추천한다. 

### 📌 서버 설정 (Tomcat)

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/13cf5bf6-68bc-4dca-936b-014c82ac035d){:width="50%"}

파일 - 설정 - 플러그인 - Tomcat 검색 - Smart Tomcat 설치

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/61157b2d-e5ac-406e-ab97-d6315044d86a)

우측 상단 - 구성 편집 

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/303f1ace-c3c0-42b9-837a-64bb9edea715)

좌측 - 새 항목 추가 - 설치한 'Smart Tomcat'

1. 설치한 Tomcat 폴더 위치 선택 

만약 톰켓을 설치하고 처음 등록하면 에러가 발생할 수 있는데 해당 폴더에 접근 권한이 없어서일 가능성이 높다. 

파일 탐색기를 실행해 직접 Tomcat을 설치한 위치를 열어 권한을 부여받고 다시 경로를 설정하면 해결된다. 

2. Context Path

해당 프로젝트 서버 실행 시에 따라다니는 경로로 "/"로 설정하면 아무것도 없는 경로로 설정된다. 원하는대로 설정해주면 된다. 

3. VM options

"-Dfile.encoding=UTF-8" 입력 (한글 깨짐 방지)

4. 포트 설정 

다른 프로젝트에서 설정한 포트와 충돌이 일어나지 않도록 설정




*** 



***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}