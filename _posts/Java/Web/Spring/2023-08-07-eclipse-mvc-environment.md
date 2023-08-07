---
title:  "[Web/IDE] Eclipse에서 Spring(MVC)환경 구축"  
excerpt: "Eclipse 환경에서 Spring 구축하기"

categories: # 분류하고싶은 카테고리 입력
  - Spring
tags:     # 중요 키워드
  - [Eclipse, STS]

toc: true
toc_sticky: true
published: true

author: Hwet

date: 2023-08-07
last_modified_at: 2023-08-07
---

인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.  
[참고]()
{: .notice--warning}


## Eclipse에서 Spring MVC Web 환경 프로젝트 구축하기

이클립스는 Spring 환경설정을 제공하기 때문에 IntelliJ보다 손쉽게 생성할 수 있다. 

### 📌 프로젝트 환경

Eclipse(2020-03 / 4.15.0), Mysql(8.0.34), JDK(8). Apache Tomcat(9.0)

### 📌 STS (Spring Tool Suite) 설치

[스프링 홈페이지](https://spring.io/) 접속 

상단 메뉴에서 Projects - DEVELOPMENT TOOLS - Spring Tools 4 클릭

Spring Tools 4 for Eclipse 태그를 찾아 해당하는 환경에 맞게 다운로드를 진행한다. 

하지만, 진행하고 있는 이클립스 환경이 4.15버전/JDK 8 버전이므로 예전 버전을 설치해야하는데 해당 홈페이지 에서 스크롤을 아래로 내리다 보면 `Spring Tool Suite 3 wiki` 버튼을 찾을 수 있다.

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/ab54134f-601e-4438-8b11-a091f4ffd2be)

버튼을 클릭하면 github홈페이지가 실행될건데 여기서 'Spring Tool Suite 3.9.14' 의 'full distribution on Eclipse 4.15'를 다운로드 한다.

파일이 다운로드가 완료되면 압축해제 후 저장하고싶은 위치로 옮겨준다. 

압축해제된 폴더 안을 확인하면 sts-3.9.14.RELEASE(설치한 버전) 폴더가 있다. 이 폴더에 들어가 보면 STS.exe 실행파일이 있는데 이 파일을 실행

실행 후 기다리면 이클립스가 실행되며 workspace 설정을 하라고 나온다. 각자 workspace에 설정한다. (이 실행 파일이 Spring 이클립스를 실행하는 파일이므로 바로가기를 만들어 바탕화면에 두고 사용하면 좋을듯 하다)

### 📌 서버 설정

STS에는 기본적으로 서버를 설치하지 않아도 사용가능 하도록 내장되어있는 기능이 있으나, 여기서는 Apache Tomcat을 사용하도록 하겠다.

'Servers'에서 우클릭 'New - Server' 클릭 'Apache - Tomcat v9.0 Server' 선택 후 'Next' 경로는 `로컬 컴퓨터 내 설치된 Tomcat 폴더` JRE 설정은 `jre-1.8` 로 설정했다.

`모든 설정 정보는 위에서 언급한 프로젝트 환경에 따라 작성하고 있으며, 다른 버전을 사용중이면 그에 맞는 정보를 입력`

### 📌 프로젝트 생성

'File - New - Other - Spring - Spring Legacy Project' 클릭

Project name : `프로젝트 이름`

Templates : `Spring MVC Project`

설정 후 'Next' 버튼을 클릭하면 Please specify the top-level package e.g com.mycompany.myapp 로 되어있는 입력창이 나온다. 

예시에 나온것 처럼 일반적으로 실제 운용되는 홈페이지의 순서를 반대로 작성하는데 실제로 운영되는 홈페이지가 없으므로 임의로 작성하도록 한다. 

`com.codingrecipe.project`로 설정하였으며 반드시 가장 뒤에 작성되는 정보는 사용될 프로젝트의 아티팩트 ID값이다.

이 과정에서 해당 프로젝트를 처음 생성할 경우 또는 추가적으로 사용되는 정보가 있는 경우에 특정 용량이 사용될것이라고 확인창이 나오는데 'Yes'를 눌러주면 된다.

### 📌 프로젝트 구조 확인

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

프로젝트 생성 시 resources 폴더가 여러개 있으므로 헷갈리지 않고 사용하여야 한다.

### 📌 프로젝트 구조 변경

> pom.xml

```xml
<properties>
		<java-version>1.8</java-version>
		<org.springframework-version>3.1.1.RELEASE</org.springframework-version>
		<org.aspectj-version>1.6.10</org.aspectj-version>
		<org.slf4j-version>1.6.6</org.slf4j-version>
	</properties>
	<dependencies>
		<!-- Spring -->
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-context</artifactId>
			<version>${org.springframework-version}</version>
			<exclusions>
				<!-- Exclude Commons Logging in favor of SLF4j -->
				<exclusion>
					<groupId>commons-logging</groupId>
					<artifactId>commons-logging</artifactId>
				 </exclusion>
			</exclusions>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-webmvc</artifactId>
			<version>${org.springframework-version}</version>
		</dependency>
		
		<!-- AspectJ -->
		<dependency>
			<groupId>org.aspectj</groupId>
			<artifactId>aspectjrt</artifactId>
			<version>${org.aspectj-version}</version>
		</dependency>	
		
		<!-- Logging -->
		<dependency>
			<groupId>org.slf4j</groupId>
			<artifactId>slf4j-api</artifactId>
			<version>${org.slf4j-version}</version>
		</dependency>
		<dependency>
			<groupId>org.slf4j</groupId>
			<artifactId>jcl-over-slf4j</artifactId>
			<version>${org.slf4j-version}</version>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>org.slf4j</groupId>
			<artifactId>slf4j-log4j12</artifactId>
			<version>${org.slf4j-version}</version>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>log4j</groupId>
			<artifactId>log4j</artifactId>
			<version>1.2.15</version>
			<exclusions>
				<exclusion>
					<groupId>javax.mail</groupId>
					<artifactId>mail</artifactId>
				</exclusion>
				<exclusion>
					<groupId>javax.jms</groupId>
					<artifactId>jms</artifactId>
				</exclusion>
				<exclusion>
					<groupId>com.sun.jdmk</groupId>
					<artifactId>jmxtools</artifactId>
				</exclusion>
				<exclusion>
					<groupId>com.sun.jmx</groupId>
					<artifactId>jmxri</artifactId>
				</exclusion>
			</exclusions>
			<scope>runtime</scope>
		</dependency>

		<!-- @Inject -->
		<dependency>
			<groupId>javax.inject</groupId>
			<artifactId>javax.inject</artifactId>
			<version>1</version>
		</dependency>
				
		<!-- Servlet -->
		<dependency>
			<groupId>javax.servlet</groupId>
			<artifactId>servlet-api</artifactId>
			<version>2.5</version>
			<scope>provided</scope>
		</dependency>
		<dependency>
			<groupId>javax.servlet.jsp</groupId>
			<artifactId>jsp-api</artifactId>
			<version>2.1</version>
			<scope>provided</scope>
		</dependency>
		<dependency>
			<groupId>javax.servlet</groupId>
			<artifactId>jstl</artifactId>
			<version>1.2</version>
		</dependency>
	
		<!-- Test -->
		<dependency>
			<groupId>junit</groupId>
			<artifactId>junit</artifactId>
			<version>4.7</version>
			<scope>test</scope>
		</dependency>    
		
      <!-- https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-databind -->
      <dependency>
         <groupId>com.fasterxml.jackson.core</groupId>
         <artifactId>jackson-databind</artifactId>
         <version>2.14.2</version>
      </dependency>
      <dependency>
         <groupId>net.sf.json-lib</groupId>
         <artifactId>json-lib</artifactId>
         <version>2.4</version>
         <classifier>jdk15</classifier>    
      </dependency>
      
		<!-- tiles라이브러리 : 화면Layout템플릿 -->
      <dependency>
         <groupId>org.apache.tiles</groupId>
         <artifactId>tiles-core</artifactId>
         <version>2.2.2</version>
      </dependency>
      <dependency>
         <groupId>org.apache.tiles</groupId>
         <artifactId>tiles-jsp</artifactId>
         <version>2.2.2</version>
      </dependency>
      <dependency>
         <groupId>org.apache.tiles</groupId>
         <artifactId>tiles-servlet</artifactId>
         <version>2.2.2</version>
    </dependency> 
    
    <!-- mysql연동 -->
      <!-- https://mvnrepository.com/artifact/mysql/mysql-connector-java -->
      <dependency>
          <groupId>mysql</groupId>
          <artifactId>mysql-connector-java</artifactId>
          <version>8.0.24</version>
      </dependency>
      
      <!-- JDBC 연동을 위한 라이브러리  -->
      <!-- spring-jdbc : 커넥션 풀 설치를 위한 라이브러리 -->
      <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-jdbc</artifactId>
            <version>${org.springframework-version}</version>
       </dependency>
       <!-- commons-dbcp2 : 실제 커넥션 풀을 처리하기 위한 라이브러리 -->
       <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-dbcp2</artifactId>
            <version>2.9.0</version>
       </dependency>
      
      <!-- 데이터소스 관련 라이브러리 등록 -->
      <!-- https://mvnrepository.com/artifact/commons-beanutils/commons-beanutils -->
      <dependency>
          <groupId>commons-beanutils</groupId>
          <artifactId>commons-beanutils</artifactId>
          <version>1.9.4</version>
      </dependency>
      <dependency>
         <groupId>commons-dbcp</groupId>
         <artifactId>commons-dbcp</artifactId>
         <version>1.2.2</version>
      </dependency>
      <dependency>
         <groupId>cglib</groupId>
         <artifactId>cglib-nodep</artifactId>
         <version>2.2</version>
      </dependency>
    
    
	<!-- mybatis관련 라이브러리 등록 -->
      <!-- mybatis : mybatis를 위한 라이브러리 -->
      <dependency>
         <groupId>org.mybatis</groupId>
         <artifactId>mybatis</artifactId>
         <version>3.1.0</version>
      </dependency>
      <!-- mybatis-spring : mybatis와  spring를 연결 라이브러리  -->
      <dependency>
         <groupId>org.mybatis</groupId>
         <artifactId>mybatis-spring</artifactId>
         <version>1.1.0</version>
      </dependency>
      
      <!-- 오라클연동 라이브러리 (이클립스는 오라클을 자동 지원하지 않으므로 시스템에 설치된 정보를 사용해야함 - 직접 설정) -->
      <!-- <dependency>
          <groupId>oracle.jdbc</groupId>
          <artifactId>OracleDriver</artifactId>
          <version>11.2.0</version>
          <scope>system</scope>
          시스템에 설치된 oracle을 사용한다는 의미
          <systemPath>${basedir}/src/main/webapp/WEB-INF/lib/ojdbc6.jar</systemPath>
      </dependency> -->
		    
	</dependencies>
    <build>
        <plugins>
            <plugin>
                <artifactId>maven-eclipse-plugin</artifactId>
                <version>2.9</version>
                <configuration>
                    <additionalProjectnatures>
                        <projectnature>org.springframework.ide.eclipse.core.springnature</projectnature>
                    </additionalProjectnatures>
                    <additionalBuildcommands>
                        <buildcommand>org.springframework.ide.eclipse.core.springbuilder</buildcommand>
                    </additionalBuildcommands>
                    <downloadSources>true</downloadSources>
                    <downloadJavadocs>true</downloadJavadocs>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>2.5.1</version>
                <configuration>
                    <source>1.6</source>
                    <target>1.6</target>
                    <compilerArgument>-Xlint:all</compilerArgument>
                    <showWarnings>true</showWarnings>
                    <showDeprecation>true</showDeprecation>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>exec-maven-plugin</artifactId>
                <version>1.2.1</version>
                <configuration>
                    <mainClass>org.test.int1.Main</mainClass>
                </configuration>
            </plugin>
        </plugins>
    </build>
```

설정 환경에 필요한 라이브러리를 추가해주면 되며, IntelliJ와 다르게 저장하면 자동으로 다운 받는다. 

> web.xml

```xml 
  <filter>
      <filter-name>encodingFilter</filter-name>       
      <filter-class>org.springframework.web.filter.CharacterEncodingFilter
      </filter-class>
      <init-param>
         <param-name>encoding</param-name>
         <param-value>UTF-8</param-value>
      </init-param>
   </filter>
   <filter-mapping>
      <filter-name>encodingFilter</filter-name>
      <url-pattern>/*</url-pattern>
   </filter-mapping>
```

한글처리를 위해서 web.xml에 추가해준다. 

### 📌 실행 

프로젝트 생성시 설정했던 기본 패키지를 열어보면 'HomeController' 클래스가 존재하는데, 서버 실행 시 가장 기본 페이지를 열어주도록 설장하는 파일이다.

return값에 "home"이라고 적혀있는데, 기본 설정값으로 지정된 위치에 있는 파일 경로를 읽어 파일을 실행시켜 브라우저에 출력해준다.

`"src/main/webapp/WEB-INF/spring/appServlet/servlet-context.xml"에 설정된 값을 확인하면 controller, views의 기본 경로를 확인할 수 있다.`

서버는 톰켓을 사용하기로 했기 때문에 서버 실행시 톰켓을 선택해주며, 생성한 프로젝트를 실행될 서버에 추가하면 브라우저에서 "home.jsp"가 실행됨을 확인할 수 있다.



*** 



***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}