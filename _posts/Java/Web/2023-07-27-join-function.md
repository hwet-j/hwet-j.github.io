---
title:  "[Web] 회원가입"  
excerpt: "회원가입 기능 구현/유효성 검사"

categories: # 분류하고싶은 카테고리 입력
  - Web
tags:     # 중요 키워드
  - [HTML, JSP, AJAX]

toc: true
toc_sticky: true
published: true

author: Hwet

date: 2023-07-27
last_modified_at: 2023-08-05
---

회원가입관련 기본 정보는 뽐뿌 홈페이지의 형식을 참고하여 작성하였습니다. 
[뽐뿌](https://www.ppomppu.co.kr/)
{: .notice--warning}


## 회원가입

### 사용 기술 

- HTML
- CSS
- JS
- JQUERY
- JAVA
- AJAX
- MySQL

### MySQL 회원 테이블 생성

```sql 
CREATE TABLE IF NOT EXISTS user_info (
  user_id         VARCHAR(20) PRIMARY KEY,      -- 회원아이디
  user_pw         VARCHAR(16) NOT NULL,         -- 비밀번호
  user_name       VARCHAR(20),                  -- 이름
  user_birth      DATE,                         -- 생일
  user_nickname   VARCHAR(40) UNIQUE,           -- 닉네임
  user_gender     ENUM('남성', '여성'),           -- 성별 
  user_tlno       VARCHAR(20) UNIQUE,           -- 핸드폰번호
  user_joindate   DATE DEFAULT (CURRENT_DATE)   -- 회원가입날짜 
);
```

> 테이블의 특징 및 작성 이유

필수입력 요소는 user_id, user_pw 두 개뿐이며, 관리자용 또는 테스트용 아이디를 간편하게 만들어 사용하기 위해서 설정

생일 및 가입날짜는 시간데이터까지는 필요없다고 판단하여 DATETIME 타입이 아닌 DATE 타입으로 설정

성별 정보는 2가지 뿐이므로 MySQL의 데이터 타입중 하나인 ENUM을 사용하여 정의한 목록들로만 입력되도록 설정

### 회원가입 폼 생성

`뽐뿌 페이지의 회원가입 형식을 가져와 사용할 DB에 맞게 수정하였습니다.`  [뽐뿌페이지이동](https://www.ppomppu.co.kr/)

![회원가입폼](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/9f1e6057-b5af-4f74-8abf-ae8782bd8f8e)

> 뽐뿌 회원가입 페이지의 특징 

- 각 입력요소에 focus 이벤트가 발생하면, 해당 요소의 작성 규칙 안내 말풍선이 생긴다.
- 각 입력요소에서 blur 이벤트가 발생하면, 해당 요소의 작성 규칙 안내 말풍선이 사라지면서 유효성 검사를 진행한다.
- "가입하기" 버튼을 클릭하면, 모든 요소의 유효성 검사가 올바르게 설정되었는지 확인한다.
- 만약 유효성 검사가 올바르게 진행되지 않았다면 각 요소에 경고문이 작성되며, 가입진행이 불가능하다.
- 모든 정보를 규칙에 맞게 작성하였으면, 최종적으로 중복검사를 실행한다.

### 입력요소 규칙 및 말풍선 설정

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/543d20e4-9ce5-4edc-9855-8f02655d6650){:width="50%"}

`이미지 내에서 아이디 입력요소를 확인`해보면 focus 이벤트가 발생하면 해당 입력요소에 맞는 규칙 말풍선이 표시되며, blur 이벤트가 발생하면 말풍선이 사라지고 규칙에 맞는지 유효성 검사를 실시힌다.

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/9490321b-ef34-41f8-b8d0-a1c2ea3c887a){:width="50%"}



#### 말풍선 표시(focus)

ID, PASSWORD의 규칙 및 말풍선은 참고한 정보의 대부분을 그대로 사용 이외에 규칙이 필요해 보이는 정보는 규칙을 새로 작성

``` 
$('#user_id').focus(function(){
		var id_info_offset = $("#user_id").offset();
		var id_info_width = $("#user_id").width();
		var id_info_top = id_info_offset.top - 35;
		var id_info_left = id_info_offset.left + id_info_width + 20;
		$("#id_info_span").css("top", id_info_top).css("left", id_info_left).show();
	});
```

- offset() : 선택한 요소의 위치정보를 반환한다. (화면 좌측상단을 기준으로 위치값을 얻어올 수 있다. top과 left의 속성을 얻어온다.)
- width() : 선택한 요소의 너비 값을 반환한다. (픽셀값으로 가로의 길이를 반환함)
- css("top", id_info_top) : css의 top 속성의 값을 입력한 변수의 값으로 설정 : 세로 위치조정
- css("left", id_info_left) : css의 left 속성의 값을 입력한 변수의 값으로 설정 : 가로 위치조정
- show() : 선택한 요소를 화면에 표시합니다. 이 메서드를 활용하여 요소를 보이도록 설정가능하다.

JQUERY를 사용하여 id='user_id'의 정보를 불러와 focus 이벤트가 발생하면 실행되는 함수이다.  
해당 태그의 위치를 가져와 말풍선의 위치를 설정하고 show()를 통하여 표시해준다.  
이 작업을 하기전에 show()해줄 정보를 사전에 작성해두어야 하며, 브라우저 최초 실행시 보이지 않도록 설정되어있어야 한다. 

$("#id_info_span").hide(); 를 스크립트문에 작성해주면 보이지 않는다. 

``` 
<div class="arrow_box" id="id_info_span">
    <span>* 아이디 생성 규칙 
    <br> 1. 영문,숫자,-,_ 사용 가능 
    <br> 2. 첫 글자, 마지막 글자는 영어와 숫자만 가능
    <br> 3. 숫자와 -의 조합으로 아이디 생성 불가
    </span>
</div>
```

이렇게 하면 최종적으로 focus 이벤트가 발생하면 해당 정보가 브라우저에서 표시된다.

#### 말풍선 숨기기(blur)

``` 
$('#user_id').blur(function(){
		
		$("#id_info_span").hide();
		
		var id = $('#user_id').val().toLowerCase();
		$('#user_id').val(id);

		var result_obj = chkUserId(id);
		
		idChkResult = result_obj.flag;

		$("#id_message_span").text(result_obj.msg);
});
```

- $('#user_id').val().toLowerCase() : 아이디를 소문자로 받아와 id변수에 변환된 소문자 값을 저장 (아이디에 대문자 사용불가 - 대문자 작성시 소문자소 변경됨)
- chkUserId(id) : 함수를 불러와 실행, id를 매개변수로 사용
- hide() : 선택한 요소를 화면에서 숨깁니다. 이 메서드를 활용하여 요소를 보이지않도록 설정가능하다.
- $("#id_message_span").text(result_obj.msg) : 실행된 함수에서 가져온 정보를 해당 요소에 text로 설정하여 출력해준다. 

chkUserId 라는 함수에서 유효성 검사를 실시하여 어떤 규칙에서 문제가 발생하였는지 msg에 담아 반환하여 해당 메시지를 브라우저에 출력한다. 

### 유효성 검사 

``` 
function chkUserId(user_id){
	var obj = {
			flag : false,
			msg : ''
	};
	
	if(!user_id){
		obj.msg = '아이디를 입력해주세요.';
	}else if(user_id.length < 4 || user_id.length > 20){
		obj.msg = '4~20자를 입력해주세요.';
	}else if(user_id.indexOf("admin") > -1){
		obj.msg = 'admin이 포함된 아이디는 사용할 수 없습니다.';
	}else{ 
		
		var id_first_regType = /^[a-z0-9]*$/;
		var id_regType = /^[a-z0-9-_]*$/;
		var id_onlynumber_regType = /^[0-9-]*$/;
		
		var id_first_index = user_id.substring(0,1); 
		var id_last_index = user_id.substring( user_id.length - 1, user_id.length);
		
		if(!id_first_regType.test(id_first_index)){
			obj.msg = '첫 글자는 영문 혹은 숫자를 입력해주세요.';
		}else if(!id_regType.test(user_id)){
			obj.msg = '영문,숫자,-,_ 의 조합으로 아이디 생성이 가능합니다.';
		}else if(!id_first_regType.test(id_last_index)){
			obj.msg = '마지막 글자는 영문 혹은 숫자를 입력해주세요.';
		}else if(id_onlynumber_regType.test(user_id)){
			obj.msg = '숫자와 -의 조합으로 아이디 생성이 불가합니다.';
		}else{
			obj.flag = true;
		}
	}
	return obj;
}
```

해당 규칙을 어겼을 때 작성되는 메시지를 보면서 해당 규칙들을 이해하면 된다. 

반환되는 obj의 flag에는 유효성검사를 통과했는지에 대해 true, false로 반환한다.

msg에는 유효성검사 실패 규칙에 따라 메시지를 설정하여 반환한다.

### Ajax를 활용하여 중복확인

모든 정보를 규칙에 맞게 작성하였으면 최종적으로 DB에 있는 정보와 비교하여 중복된 데이터가 있는지 확인한다.

``` 
//아이디 중복 확인 요청
function checkDuplicate() {
    let userid = $("input[name='user_id']");
    // AJAX 요청
    $.ajax({
        url: "/view/member/checkDuplicate.jsp",
        type: "POST",
        data: { userid: userid.val() },
        success: function(response) {
        	var message = response.replace(/\s/g, ""); // 공백 제거
            if (message === "id_duplicate") {	 	// 아이디 중복
                userid.focus();
                $("#id_message_span").text("이미 사용 중인 아이디입니다.");
                idChkResult = false;
                return;
            } else {
              // 중복 정보가 없으면 submit 진행
              document.forms['write'].submit();
            	
            }
        },
        error: function(xhr, status, error) {
            console.log(error);
        }
    });
}
```

checkDuplicate 함수가 실행되면 id값을 받아와 "POST" 형식으로 url 주소를 실행하면서 data에 입력된 정보를 넘겨준다. 

해당 코드는 jsp파일을 실행하도록 하여서 jsp파일에서 DB에 연결하여 해당 id값이 존재하는지 확인하고, 존재한다면
"message" 라는 변수에 "id_duplicate"를 담아서 반환하고, 그렇지 않으면 "not_duplicate" 를 반환하도록 설정하였다.

요청에 성공했으면 ajax의 success 부분이 실행되고, 전송받은 message는 response객체에 담겨들어온다. 
이 과정에서 줄 바꿈 정보까지 같이 넘어오는 문제가 발생하였는데 response.replace(/\s/g, "") 로 공백을 제거하여 해결하였다. 

message를 확인하여 중복인지 아닌지 확인해 중복이면 id_message_span에 텍스트 문구를 설정해주고, 중복이 아니면 submit을 진행한다. 

`jsp 파일로 ajax를 실행하는 것은 추천하지 않는 방식이라고 하지만, 아직 java파일로 실행하는 방법을 공부하지 않아서 jsp로 진행하였다. (보안상 좋지 않다고 함)`








*** 



***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}