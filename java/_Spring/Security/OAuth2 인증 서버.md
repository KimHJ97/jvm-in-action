# OAuth2 인증 서버

OAuth2 인증 흐름에서 중앙 역할을 하는 서버

 - 사용자 인증(Authentication)
 - 클라이언트 인증(Client Authentication)
 - Access Token 발급
 - Refresh Token 발급
 - 권한 승인 (Authorization Consent)

## 1. spring-boot-starter-oauth2-authorization-server

spring-security-oauth2-authorization-server 모듈은 OAuth 2.0 인증 서버(Authorization Server)를 구축하기 위한 Spring Security의 공식 지원 라이브러리입니다. 클라이언트 애플리케이션이 액세스 토큰(Access Token)과 리프레시 토큰(Refresh Token)을 발급받을 수 있도록 인증 및 권한 부여 엔드포인트를 제공합니다.

 - Authorization Code Grant
 - Client Credentials Grant
 - Refresh Token Grant
 - PKCE 지원
 - OIDC (OpenID Connect 1.0) 지원
 - JWK Set endpoint (/.well-known/jwks.json)
 - Token Introspection endpoint (/oauth2/introspect)
 - Token Revocation endpoint (/oauth2/revoke)
 - Consent 화면 제공
 - Client 등록 (DB 또는 In-Memory)

### 1-1. 주요 엔드포인트

```
/oauth2/authorize:      인증 코드 발급 요청
/oauth2/token:          토큰 발급 요청
/oauth2/jwks:           공개키 제공 (JWT 검증용)
/.well-known/jwks.json: 공개키 제공 (JWT 검증용)
/oauth2/introspect:     액세스 토큰 유효성 검증 (opaque용)
/oauth2/revoke:         토큰 폐기 요청
/oauth2/userinfo:       사용자 정보 제공 (OIDC)
/login:                 사용자 로그인 화면
/consent:               권한 동의 화면
```

### 1-2. 설정 방법

 - `build.gradle`
```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-authorization-server'
}
```
