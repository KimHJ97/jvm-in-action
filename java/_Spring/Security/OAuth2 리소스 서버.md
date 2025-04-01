# OAuth2 리소스 서버

OAuth2에서 Resource Server는 보호된 리소스를 가지고 있으며, 클라이언트가 Access Token을 통해 요청할 수 있는 서버입니다.

 - 백엔드 API 서버가 Resource Server가 되고, 토큰은 인증 서버(Authorization Server)에서 발급합니다.
 - Resource Server는 이 토큰을 검증하고, 사용자 정보를 추출하거나 접근 권한을 체크합니다.

## 1. spring-boot-starter-oauth2-resource-server

spring-boot-starter-oauth2-resource-server는 Spring Security에서 OAuth2 기반 Resource Server를 구현할 때 사용하는 스타터 모듈입니다. 이 모듈은 JWT 또는 Opaque Token을 사용한 인증을 처리하며, 외부 인증 서버(Authorization Server)와 연동하여 API 요청에 대한 접근 제어를 수행할 수 있게 해줍니다.

 - __Access Token 검증__
    - JWT 또는 Opaque Token 기반 토큰을 검증합니다.
    - 토큰 검증을 위해 JWK Set URI 또는 introspection endpoint를 설정합니다.
 - __인증된 사용자 정보 추출__
    - JWT Claims를 Authentication 객체에 자동으로 매핑합니다.
    - 권한 (scope, authorities) 정보도 SecurityContext에 자동으로 담깁니다.
 - __HTTP 요청 보호__
    - @PreAuthorize, @Secured, SecurityFilterChain 등을 통해 API 접근 제어 설정이 가능합니다.

### 1-1. 기본 동작 방식

 - 클라이언트가 Authorization Server에서 Access Token을 받음.
 - 클라이언트가 Resource Server(Spring Boot API 서버)에 토큰을 포함하여 요청.
 - Resource Server는 해당 토큰을 검증 (iss, aud, exp, signature 등).
 - 검증에 성공하면, SecurityContext에 인증 정보가 설정됨.
 - 이후 인증된 사용자로 요청을 처리함.

### 1-2. 설정 방법

 - `build.gradle`
```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
}
```
