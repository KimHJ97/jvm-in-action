# OAuth2 클라이언트

 - OAuth 2.0 인증 서버와 연동하여 사용자 인증을 처리
 - Authorization Code Grant 방식을 기본 지원
 - 구글, 네이버, 카카오 등 소셜 로그인 구현
 - OAuth2 인증 이후 사용자 정보를 가져오고 세션에 저장
 - Spring Security와 통합하여 보안 적용

## 1. spring-boot-starter-oauth2-client

spring-boot-starter-oauth2-client는 Spring Security에서 OAuth2 클라이언트 기능을 쉽게 사용할 수 있도록 도와주는 Starter 모듈입니다. 주로 소셜 로그인이나 외부 인증 서버를 통해 사용자 인증을 하고 싶은 경우에 사용됩니다.

 - OAuth2 Client: 외부 OAuth2 인증 서버에 클라이언트로 동작
 - Authorization Code Flow: 인증 코드 흐름으로 사용자 인증 처리
 - OpenID Connect (OIDC): OIDC 인증 서버와 연동 가능 (id_token 사용)
 - 사용자 정보 매핑: 인증 후 사용자 정보(OAuth2User) 매핑 가능
 - 자동 로그인 처리: 인증 성공 시 자동으로 Security Context에 저장

### 1-1. OAuth2 로그인 처리 방식

 - /oauth2/authorization/{registrationId}: 로그인 진입점
    - /oauth2/authorization/google
    - /oauth2/authorization/naver
 - 인증 완료 후 OAuth2UserService를 통해 사용자 정보 로딩

### 1-2. 설정 방법

 - `build.gradle`
```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-web'
}
```

 - `application.yml`
```yml
spring:
  security:
    oauth2:
      client:
        registration:
          kakao:
            client-id: [REST_API_KEY]
            client-secret: [CLIENT_SECRET]
            client-name: Kakao
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            scope:
              - profile_nickname
              - account_email
        provider:
          kakao:
            authorization-uri: https://kauth.kakao.com/oauth/authorize
            token-uri: https://kauth.kakao.com/oauth/token
            user-info-uri: https://kapi.kakao.com/v2/user/me
            user-name-attribute: id
```

