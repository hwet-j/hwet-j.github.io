---
title:  "[Web] Spring Boot에서 Kakao API를 사용하여 로그인 구현하기 (Oauth2)"  
excerpt: "API를 활용한 로그인 기능"

categories: # 분류하고싶은 카테고리 입력
  - Spring
tags:     # 중요 키워드
  - [KakaoAPI]

toc: true
toc_sticky: true
published: true

author: Hwet

date: 2023-08-21
last_modified_at: 2023-08-21
---

인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.  
[참고](https://bcp0109.tistory.com/379)
[참고](https://github.com/codingspecialist/-Springboot-Security-OAuth2.0-V3)
[참고](https://loosie.tistory.com/302)
{: .notice--warning}


# 수정 진행 중입니다.


## Kakao API를 사용하여 로그인/회원가입 하기 (Oauth2)

작동원리나 이론적인 내용은 완벽하게 이해하지 못한 상태에서 구현에만 초점을 두고 작성하고 진행하였습니다.

이후에 작동원리나 이론을 이해하면 따로 정리하도록 하겠습니다. 


### 개발환경

Spring boot 3.0.10  
Gradle  
JAVA 17  
Thymeleaf  
Lombok  
Spring Web  
Spring Data JPA  
Spring Security  
Spring Oauth2-Client

### Kakao API 등록하기

> 카카오 개발자 홈페이지 접속

[카카오 개발자 페이지](https://developers.kakao.com/)

> "카카오 로그인"

자신의 카카오 아이디로 로그인 합니다.

> "내 어플리케이션" 이동

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/1128f923-29f5-46f9-9190-679ab16c324f)

> "어플리케이션 추가하기"

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/ea90327b-cb51-4621-a219-b4ec7505c91c)

사업자가 아니더라도 적절한 이름 설정하여 생성가능

> 사이트 도메인 등록

`앱 설정 -> 플랫폼 -> Web -> Web 플랫폼 등록`

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/99ac57b1-7e31-4205-bb3a-6d793c225a65)

사용할 URL을 설정하면 되며, 테스트로 사용될 것이니 IP주소는 localhost로 설정하고 포트 번호는 사용할 포트번호로 작성해주고 저장한다.

Ex) <http://localhost:8080>

> 카카오 로그인 활성화 하기 

`제품 설정 -> 카카오 로그인 -> 활성화 설정 -> ON`

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/cc9474ed-e0f2-4813-9575-20947c730fb2)

여기서 추가적으로 Redirect URI를 등록해줘야한다. (<http://localhost:8080/login/oauth2/code/kakao>)

> 동의항목 설정하기

`동의 항목 -> 개인정보 설정 -> 필수/선택 동의 체크 -> 동의 목적 입력`

필요 항목에 따라 설정해주며 이메일을 입력 받기 위해서는 검수가 별도로 필요하다. 

> 검수받기

`앱설정 -> 일반 -> 기본 정보 -> 수정`

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/9a6ae764-34d2-426e-a84e-401d5be2df30)

앱 아이콘을 설정한다. 해당 아이콘은 동의 화면에서 출력되는 이미지

`앱설정 -> 비즈니스 -> 개인 개발자 비즈앱 전환 -> 이메일 필수 동의 -> 전환`

![image](https://github.com/hwet-j/hwet-j.github.io/assets/81364742/51b380eb-200a-43a9-96a0-5785e753d6f6)

앱 아이콘을 설정하지 않으면 '개인 개발자 비즈앱 전환'이 보이지 않으므로 보이지 않으면 아이콘이 정상적으로 설정되었는지 확인

이메일을 필수동의로 사용할거라면 등록이 완료 후 다시 동의항목 설정으로 돌아가 이메일을 필수동의로 변경한다.

### Spring Boot 설정 

#### application.yml

```
spring:
  security:
    oauth2:
      client:
        registration:
          kakao:
            client-id: 클라이언트 ID (REST API 키)
            client-authentication-method: client_secret_post
            redirect-uri: Redirect URI
            client-name: kakao
            scope:
              - account_email
            authorization-grant-type: authorization_code

        provider:
          kakao:
            authorization-uri: https://kauth.kakao.com/oauth/authorize
            token-uri: https://kauth.kakao.com/oauth/token
            user-info-uri: https://kapi.kakao.com/v2/user/me
            user-name-attribute: id
```

카카오는 구글과 다르게 Spring에서 Oauth2를 지원해주지 않기 때문에 Provider를 직접입력하여 설정한다.


#### SecurityConfig.java

```java 
@Configuration
@RequiredArgsConstructor
@EnableWebSecurity
public class SecurityConfig {

	private final PrincipalOauth2UserService principalOauth2UserService;

	@Bean
	public BCryptPasswordEncoder encodePwd() {
		return new BCryptPasswordEncoder();
	}

	@Bean
	public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {

		http
			.authorizeRequests((authorizeHttpRequests) -> authorizeHttpRequests
					.requestMatchers(new AntPathRequestMatcher("/**")).permitAll()
					.requestMatchers(new AntPathRequestMatcher("/admin/**")).access("hasRole('ROLE_ADMIN')"))
			.headers((headers) -> headers.addHeaderWriter(new XFrameOptionsHeaderWriter(
					XFrameOptionsHeaderWriter.XFrameOptionsMode.SAMEORIGIN)))
			.formLogin((formLogin)->formLogin.loginPage("/login")
					.loginProcessingUrl("/loginProc")
					.defaultSuccessUrl("/"))
			.oauth2Login(oauth2Login -> oauth2Login
					.loginPage("/login")
					.userInfoEndpoint(userInfo -> userInfo
							.userService(principalOauth2UserService))
			).csrf(csrf -> csrf.disable());

		return http.build();
	}
}
```

``` 
# Oauth2 설정 부분
.oauth2Login(oauth2Login -> oauth2Login
					.loginPage("/login")
					.userInfoEndpoint(userInfo -> userInfo
							.userService(principalOauth2UserService))
			)
```

일반로그인 이외의 별도의 Oauth2Login 설정을 해줘야한다. 

`loginPage`의 설정은 로그인 페이지를 설정하는 부분이며 일반로그인 페이지와 동일하게 설정 

`userInfoEndpoint`은 인증 후 사용자 정보를 어떻게 처리할지 설정하는 부분

`userService`는 사용자 정보를 가져오는 서비스를 설정하며, 사용자 정보를 가져오는데 사용될 객체는 `principalOauth2UserService`로 설정한 것이다.

#### PrincipalOauth2UserService.java

```java 
@Service
@Slf4j
@RequiredArgsConstructor
public class PrincipalOauth2UserService extends DefaultOAuth2UserService {

	private final UserRepository userRepository;

	@Override
	public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
		OAuth2User oAuth2User = super.loadUser(userRequest);

		return processOAuth2User(userRequest, oAuth2User);
	}

	private OAuth2User processOAuth2User(OAuth2UserRequest userRequest, OAuth2User oAuth2User) {

		OAuth2UserInfo oAuth2UserInfo = null;
		if (userRequest.getClientRegistration().getRegistrationId().equals("google")) {
			log.info("구글 로그인 요청");
			oAuth2UserInfo = new GoogleUserInfo(oAuth2User.getAttributes());
		} else if (userRequest.getClientRegistration().getRegistrationId().equals("kakao")) {
			log.info("카카오 로그인 요청");
			oAuth2UserInfo = new KakaoUserInfo(oAuth2User.getAttributes());
		} else if (userRequest.getClientRegistration().getRegistrationId().equals("naver")){
			log.info("네이버 로그인 요청");
			oAuth2UserInfo = new NaverUserInfo((Map)oAuth2User.getAttributes().get("response"));
		} else {
			log.warn("구글,카카오,네이버 이외의 로그인 요청");
		}

		Optional<User> userOptional =
				userRepository.findByProviderAndProviderId(oAuth2UserInfo.getProvider(), oAuth2UserInfo.getProviderId());

		long countUser = userRepository.countByEmail(oAuth2UserInfo.getEmail());

		User user;
		if (countUser == 1) {	
			user = userRepository.findByEmail(oAuth2UserInfo.getEmail());
			log.info("회원가입 정보 존재");
		} else if (userOptional.isPresent()) {	
			user = userOptional.get();
			user.setEmail(oAuth2UserInfo.getEmail());
			userRepository.save(user);
			log.info("API 이메일 정보 변경");
		} else {   
			user = User.builder()
					.username(oAuth2UserInfo.getProvider() + "_" + oAuth2UserInfo.getProviderId())
					.email(oAuth2UserInfo.getEmail())
					.role("ROLE_USER")
					.provider(oAuth2UserInfo.getProvider())
					.providerId(oAuth2UserInfo.getProviderId())
					.build();
			userRepository.save(user);
			log.info("회원가입 정보 없음");
		}

		return new PrincipalDetails(user, oAuth2User.getAttributes());
	}
}
```

여기서 따로 구현하지 않았지만 구글, 네이버도 함께 연동할 예정이라 함께 작성되어있다. 카카오만 사용할 예정이라면 `userRequest.getClientRegistration().getRegistrationId()`의 정보를 사용하지 않아도된다.

API로 간편로그인이 가능하지만, 회원가입을 진행하여 별도의 추가정보를 입력받을 예정이므로 해당 이메일로 가입된 정보가 있는지 확인한다. `countUser`가 1이라면 가입이력이 있으므로 객체를 저장 후 리턴한다.

이메일로 회원가입한 이력이 없으나 `userOptional`이 존재한다면, 연동되어있는 이메일 값을 변경했을 가능성을 염두에 두고 이메일 값을 업데이트 한다. 

둘 다 존재하지 않으면 해당 API로 회원가입이력도 없고, 이메일로 저장된 내역도 없으므로 API를 통해 얻어지는 정보를 저장하고 리턴한다. 


#### IndexController.java

`로그인이 완료되면 요청되는 Controller`



































*** 



***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}