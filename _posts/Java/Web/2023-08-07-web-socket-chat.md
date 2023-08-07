---
title:  "[Web] 웹소켓을 활용하여 실시간 채팅 구현하기"  
excerpt: "웹소켓에 대해 알아보자"

categories: # 분류하고싶은 카테고리 입력
  - Web
tags:     # 중요 키워드
  - [HTML, JSP, AJAX, WebSocket]

toc: true
toc_sticky: true
published: false

author: Hwet

date: 2023-08-07
last_modified_at: 2023-08-07
---

인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.
[참고1](https://milkoon1.tistory.com/92)
[참고2](https://designatedroom87.tistory.com/323)
{: .notice--warning}


# 웹소켓을 활용한 채팅 기능 구현해보기

Ajax을 통해서 클라이언트가 보고있는 페이지가 실시간으로 변하기 때문에 이 기능만 사용해서 채팅기능을 구현할 수 있을 줄 알고 시작했지만,
다른 사람이 동일한 홈페이지에 접속해 있을 때, 다른 사람은 Ajax를 통한 변화를 확인할 수 없었습니다. 

Ajax는 단방향 요청-응답을 하기 때문에 요청을 보내야지만 응답이 오게된다. 
채팅 기능을 구현하기 위해 일정한 주기로 요청을 보낸다거나 버튼을 생성하여 갱신하도록 한다면 서버에 부하와 성능 저하를 초래할 수 있다. 

그렇게 웹소켓이라는 양방향 통신을 위한 프로토콜을 사용하여 채팅기능을 구현해보았다.

## ❓ 웹소켓이란?

> 작동원리 

1. 핸드쉐이킹(Handshaking) : 클라이언트가 웹소켓 연결을 요청할 때, HTTP 프로토콜을 통해 서버와 핸드쉐이킹을 수행합니다. 클라이언트가 웹소켓 연결을 요청하면 서버는 연결을 수락하고 양쪽 간의 웹소켓 프로토콜로 전환합니다.
2. 지속적인 연결 : 웹소켓은 HTTP와 달리 연결을 유지하며, 연결이 정상적으로 이루어진다면 서버와 클라이언트 간에 웹소켓 연결(TCP/IP기반)이 이루어지고 일정 시간이 지나면 HTTP연결은 자동으로 끊어집니다
3. 프로토콜 내부의 메시지 전달 : 웹소켓 프로토콜은 메시지의 종류, 길이 등을 포함하고 있어 메시지의 전송과 해석을 관리합니다.
4. 이벤트 기반 처리 : 웹소켓은 이벤트 기반 방식으로 동작합니다. 클라이언트나 서버에서 이벤트가 발생하면 해당 이벤트를 처리하고 필요한 동작을 수행합니다.

> 채팅 기능에 적합한 이유

1. 양방향 통신 : 클라이언트와 서버 사이에서 양방향으로 데이터를 주고받을 수 있습니다. 이로써 실시간 데이터 업데이트가 가능합니다.
2. 클라이언트-서버 연결 : 웹소켓은 클라이언트와 서버 간의 지속적인 연결을 유지하며 데이터를 전송합니다. 이를 통해 연결 상태를 유지하면서 실시간 데이터를 주고받을 수 있습니다.
3. 네트워크 부하 감소 : 웹소켓은 HTTP의 헤더 오버헤드를 줄여 네트워크 부하를 줄일 수 있습니다.

> 단점, 문제점

1. 커넥션 유지 비용 : 웹소켓 연결은 계속 유지되어야 하기 때문에 커넥션 유지 비용이 발생합니다. 이로 인해 서버 리소스의 소비가 높아질 수 있습니다.
2. 보안 문제 : 웹소켓 연결은 HTTP와는 다른 프로토콜을 사용하기 때문에 보안 설정이 별도로 필요합니다. 적절한 보안 설정을 하지 않을 경우, 보안 위협이 발생할 수 있습니다.
3. 구현 복잡성 : 웹소켓은 클라이언트 측에서도 구현이 필요한데, 일반적으로 이에 따른 복잡성이 발생합니다.
4. 서버 부하 관리 : 많은 클라이언트 연결이 동시에 발생할 경우, 서버 부하가 증가할 수 있습니다. 이에 따라 적절한 스케일링과 부하 분산이 필요하다

## ⚙️ 구현

### 📌 프로젝트 환경 

Eclipse (2020-03/4.15.0)

JRE 1.8

Apache Tomcat 9.0

MySQL

Dynamic Web Project

### 📌 웹소켓 테스트

> src/socket/BroadSocket.java

```java
package socket;

import java.io.IOException;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.regex.Matcher;
import java.util.regex.Pattern;
import javax.websocket.OnClose;
import javax.websocket.OnMessage;
import javax.websocket.OnOpen;
import javax.websocket.Session;
import javax.websocket.server.ServerEndpoint;

// WebSocket 호스트 설정
@ServerEndpoint("/broadsocket")
public class BroadSocket {
  // 접속 된 클라이언트 WebSocket session 관리 리스트
  private static List<Session> sessionUsers = Collections.synchronizedList(new ArrayList<>());
  // 메시지에서 유저 명을 취득하기 위한 정규식 표현
  private static Pattern pattern = Pattern.compile("^\\{\\{.*?\\}\\}");
  // WebSocket으로 브라우저가 접속하면 요청되는 함수	
  @OnOpen
  public void handleOpen(Session userSession) {
    // 클라이언트가 접속하면 WebSocket세션을 리스트에 저장한다.
    sessionUsers.add(userSession);
    // 콘솔에 접속 로그를 출력한다.
    System.out.println("client is now connected...");
  }
  // WebSocket으로 메시지가 오면 요청되는 함수
  @OnMessage
  public void handleMessage(String message, Session userSession) throws IOException {
    // 메시지 내용을 콘솔에 출력한다.
    System.out.println(message);
    // 초기 유저 명
    String name = "anonymous";
    // 메시지로 유저 명을 추출한다.
    Matcher matcher = pattern.matcher(message);
    // 메시지 예: {{유저명}}메시지
    if (matcher.find()) {
      name = matcher.group();
    }
    // 클로져를 위해 변수의 상수화
    final String msg = message.replaceAll(pattern.pattern(), "");
    final String username = name.replaceFirst("^\\{\\{", "").replaceFirst("\\}\\}$", "");
    // session관리 리스트에서 Session을 취득한다.
    sessionUsers.forEach(session -> {
      // 리스트에 있는 세션과 메시지를 보낸 세션이 같으면 메시지 송신할 필요없다.
      if (session == userSession) {
        return;
      }
      try {
        // 리스트에 있는 모든 세션(메시지 보낸 유저 제외)에 메시지를 보낸다. (형식: 유저명 => 메시지)
        session.getBasicRemote().sendText(username + " => " + msg);
      } catch (IOException e) {
        // 에러가 발생하면 콘솔에 표시한다.
        e.printStackTrace();
      }
    });
  }
  // WebSocket과 브라우저가 접속이 끊기면 요청되는 함수
  @OnClose
  public void handleClose(Session userSession) {
    // session 리스트로 접속 끊은 세션을 제거한다.
    sessionUsers.remove(userSession);
    // 콘솔에 접속 끊김 로그를 출력한다.
    System.out.println("client is now disconnected...");
  }
}
```

> WebContent/views/WebsocketTest.jsp

```jsp 
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head><title>Web Socket Example</title></head>
<body>
  <form>
    <!-- 유저 명을 입력하는 텍스트 박스(기본 값은 anonymous(익명)) -->
    <input id="user" type="text" value="anonymous">
    <!-- 송신 메시지를 작성하는 텍스트 박스 -->
    <input id="textMessage" type="text">
    <!-- 메세지를 송신하는 버튼 -->
    <input onclick="sendMessage()" value="Send" type="button">
    <!-- WebSocket 접속 종료하는 버튼 -->
    <input onclick="disconnect()" value="Disconnect" type="button">
  </form>
  <br />
  <!-- 콘솔 메시지의 역할을 하는 로그 텍스트 에리어.(수신 메시지도 표시한다.) -->
  <textarea id="messageTextArea" rows="10" cols="50"></textarea>
  <script type="text/javascript">
    // 「WebSocketEx」는 프로젝트 명
    // 「broadsocket」는 호스트 명
    // WebSocket 오브젝트 생성 (자동으로 접속 시작한다. - onopen 함수 호출)
    var webSocket = new WebSocket("ws://localhost:설정한포트/서블릿에맵핑한정보");
    // 콘솔 텍스트 에리어 오브젝트
    var messageTextArea = document.getElementById("messageTextArea");
    // WebSocket 서버와 접속이 되면 호출되는 함수
    webSocket.onopen = function(message) {
      // 콘솔 텍스트에 메시지를 출력한다.
      messageTextArea.value += "Server connect...\n";
    };
    // WebSocket 서버와 접속이 끊기면 호출되는 함수
    webSocket.onclose = function(message) {
      // 콘솔 텍스트에 메시지를 출력한다.
      messageTextArea.value += "Server Disconnect...\n";
    };
    // WebSocket 서버와 통신 중에 에러가 발생하면 요청되는 함수
    webSocket.onerror = function(message) {
      // 콘솔 텍스트에 메시지를 출력한다.
      messageTextArea.value += "error...\n";
    };
    /// WebSocket 서버로 부터 메시지가 오면 호출되는 함수
    webSocket.onmessage = function(message) {
      // 콘솔 텍스트에 메시지를 출력한다.
      messageTextArea.value += message.data + "\n";
    };
    // Send 버튼을 누르면 호출되는 함수
    function sendMessage() {
      // 유저명 텍스트 박스 오브젝트를 취득
      var user = document.getElementById("user");
      // 송신 메시지를 작성하는 텍스트 박스 오브젝트를 취득
      var message = document.getElementById("textMessage");
      // 콘솔 텍스트에 메시지를 출력한다.
      messageTextArea.value += user.value + "(me) => " + message.value + "\n";
      // WebSocket 서버에 메시지를 전송(형식 「{{유저명}}메시지」)
      webSocket.send("{{" + user.value + "}}" + message.value);
      // 송신 메시지를 작성한 텍스트 박스를 초기화한다.
      message.value = "";
    }
    // Disconnect 버튼을 누르면 호출되는 함수
    function disconnect() {
      // WebSocket 접속 해제
      webSocket.close();
    }
  </script>
</body>
</html>
```

자신이 사용중인 웹어플리케이션의 IP주소,포트번호를 잘 설정했다면 문제 없이 연결된다.

### 📌 웹소켓을 통한 기능 구현

> 구현하려는 기능

Ajax로 구현하려 했던 정보에서 수정을하여 불필요한 과정이 있을 수 있습니다.

1. Ajax를 통해서 내가 입력한 채팅 우측에 출력 
2. 채팅방 입장시 '접속하였습니다' 메시지 출력 
3. 채팅방 퇴장시 '퇴장하였습니다' 메시지 출력 
4. 상대방이 메시지 입력시 좌측에 출력
5. 여러개의 채팅방 분리
6. 채팅방 생성, 채팅방 초대

"웹소켓 테스트" 단계에서 작성해줬던 코드를 수정하여 작성하고자 하는 기능을 최대한 작성하려 했습니다. 

#### 📎 





*** 



***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}