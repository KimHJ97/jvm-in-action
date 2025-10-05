# HTTP 요청

 - https://jojoldu.tistory.com/266
 - https://jojoldu.tistory.com/366

## 1. http 파일

IntelliJ의 .http 파일은 HTTP 요청을 직접 작성하고 실행할 수 있는 테스트 스크립트 파일이다. Postman처럼 API를 테스트할 수 있지만, IDE 내부에서 버전 관리까지 가능한 개발 친화적인 방식이다.

 - Postman처럼 별도 툴 없이 테스트 가능
 - Git에 커밋 가능 (요청 이력 관리)
 - 환경변수(.env.json or .http.env.json) 지원
 - 요청 간 변수 공유 가능 ({{variable}} 문법)
 - 응답 내용 스크립트 검증 가능 (JS로 assertion)

<br/>

## 2. 사용 예시

 - `GET 요청`
```bash
### GET 요청
GET http://localhost:8080/api/users
Accept: application/json
```
<br/>

 - `POST 요청 - JSON`
```bash
### POST 요청 (JSON Body)
POST http://localhost:8080/api/login
Content-Type: application/json

{
  "email": "admin@test.com",
  "password": "1234"
}
```

 - `POST 요청 - multipart/form-data`
```bash
POST {{baseUrl}}/api/user/profile
Authorization: {{accessToken}}
Content-Type: multipart/form-data; boundary=WebAppBoundary

--WebAppBoundary
Content-Disposition: form-data; name="name"

홍길동
--WebAppBoundary
Content-Disposition: form-data; name="email"

email@test.com
--WebAppBoundary
Content-Disposition: form-data; name="age"

30
--WebAppBoundary--
```
<br/>

 - `POST 요청 - 파일 업로드`
```bash
### 파일 + JSON + 단일 텍스트 파라미터
POST http://localhost:8080/api/inquiry/save
Content-Type: multipart/form-data

--boundary
Content-Disposition: form-data; name="inquiry"
Content-Type: application/json

{
  "title": "문의드립니다",
  "content": "제품이 작동하지 않습니다."
}
--boundary
Content-Disposition: form-data; name="memberId"

12345
--boundary--
Content-Disposition: form-data; name="files"; filename="photo.jpg"
Content-Type: image/jpeg

< ./test/photo.jpg
--boundary
```
<br/>

 - `POST 요청 - 파일 업로드 (2023+)`
    - --form 문법은 IntelliJ 2023 이후 버전에서 지원
```bash
### multipart/form-data (자동 boundary)
POST http://localhost:8080/api/inquiry/save
Content-Type: multipart/form-data

--form
inquiry={
  "title": "문의드립니다",
  "content": "제품이 작동하지 않습니다."
}
--form
memberId=12345
--form
files=@photo.jpg;type=image/jpeg
--form
files=@C:/Users/User/Desktop/test2.jpg;type=image/jpeg

```
<br/>

## 3. 환경 변수

 - `http-client.env.json`
    - .http.env.json 파일을 같은 디렉토리에 만들어두면 환경별 설정을 쉽게 전환할 수 있다.
```json
{
  "dev": {
    "baseUrl": "http://localhost:8080",
    "accessToken": "devToken"
  },
  "prod": {
    "baseUrl": "https://api.myserver.com",
    "accessToken": "realToken"
  }
}
```
<br/>

 - `환경 변수 사용`
```bash
### Authorization 헤더 사용
GET {{baseUrl}}/api/profile
Authorization: Bearer {{accessToken}}
```
<br/>

## 4. 응답값 변수로 저장하고 사용하기 (여러 요청 간 변수 공유)

 - `응답 Body 추출`
```bash
### 로그인
POST {{baseUrl}}/api/v1/auth/login
Content-Type: application/json

{
  "email": "admin@example.com",
  "password": "1234"
}

> {% client.global.set("accessToken", response.body.token); %}

### 내 정보 조회
GET {{baseUrl}}/api/v1/member/me
Authorization: Bearer {{accessToken}}
```
<br/>

 - `응답 Header 추출`
```bash
### 로그인
POST {{baseUrl}}/api/v1/auth/login
Content-Type: application/json

{
  "email": "admin@example.com",
  "password": "1234"
}

> {%
client.global.set("JSESSIONID", response.headers.valueOf("Set-Cookie").split(";")[0].split("=")[1]);
client.log("쿠키: "+client.global.get("JSESSIONID"));
%}

### 글로벌 변수 사용
GET https://okky.kr/user/edit
Cookie: JSESSIONID={{JSESSIONID}}
```
<br/>

## 5. 내장 객체

요청 전후(pre/post) 에 자바스크립트 문법으로 다양한 작업을 수행할 수 있다.

 - `client`
    - client.global.set(key, value): 전역 변수 저장
    - client.global.get(key): 변수 값 가져오기
    - client.global.clear(key): 특정 변수 삭제
    - client.global.clearAll(): 모든 변수 삭제
    - client.log(message): 콘솔 출력 (디버그용)
```javascript
client.global.set("accessToken", response.body.token);
client.global.get("accessToken");
client.global.clear("accessToken");
client.log("현재 토큰:", client.global.get("accessToken"));
```
<br/>

 - `request`
    - request.variables.set(key, value): 현재 요청 범위 변수 설정
    - request.variables.get(key): 현재 요청 변수 가져오기
```javascript
request.variables.set("now", new Date().toISOString());
```
<br/>

 - `response`
    - status: HTTP 상태 코드
    - body: 응답 본문 (JSON 자동 파싱됨)
    - headers: 응답 헤더 객체
    - time: 요청-응답 소요시간(ms)
```javascript
response.status;           // 200
response.body;             // JSON, text 등
response.headers["Date"];  // 응답 헤더 접근
```
<br/>

 - `활용 예시`
```bash
### 로그인 후 토큰 저장
POST {{baseUrl}}/api/v1/auth/login
Content-Type: application/json

{
  "email": "admin@test.com",
  "password": "1234"
}

> {% 
  client.global.set("accessToken", response.body.token);
  client.log("토큰 저장 완료:", response.body.token);
%}


### 공통 헤더 추가
GET {{baseUrl}}/api/v1/member/me
> {% 
  request.headers["Authorization"] = "Bearer " + client.global.get("accessToken");
%}


### 응답 값 검증
GET {{baseUrl}}/api/v1/member/me

> {% 
  if (response.status !== 200) {
    throw new Error("응답이 200이 아님! " + response.status);
  }
  if (!response.body.email) {
    throw new Error("이메일 필드 없음!");
  }
%}


### 시간 변수 자동 생성
> {% 
  request.variables.set("timestamp", Date.now());
%}

POST {{baseUrl}}/api/v1/logs
Content-Type: application/json

{
  "createdAt": "{{timestamp}}"
}


### 복잡한 JSON 파싱
> {% 
  const devices = response.body.devices.map(d => d.name);
  client.global.set("deviceNames", devices.join(", "));
%}
```
