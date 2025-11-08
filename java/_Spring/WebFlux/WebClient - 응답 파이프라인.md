# WebClient 응답 파이프라인

## 1. 응답 파이프라인

WebClient의 응답 처리 메서드인 exchangeToFlux(), retrieve(), bodyToFlux() 등은 요청 이후의 응답 파이프라인을 어떻게 해석하고 소비할 것인지를 결정한다.

Spring WebFlux의 WebClient는 `WebClient.RequestHeadersSpec<?>` 단계까지가 __요청 준비 단계__, 그 이후부터가 __응답 처리 단계__ 이다.

<br/>

### 1-1. 저수준 API

 - __exchangeToFlux() / exchangeToMono()__
    - 가장 저수준(low-level)**의 응답 처리.
    - 응답 객체(ClientResponse) 전체를 직접 다뤄야 할 때 사용.
    - 특징
        - `ClientResponse` 를 직접 다루므로 헤더, 상태 코드 등을 세밀하게 처리 가능
        - `response.statusCode()`, `response.headers()`, `response.body(...)` 등을 직접 제어
        - `exchangeToFlux()`는 `Flux`, `exchangeToMono()`는 `Mono` 반환
        - 예외 처리(404, 500 등)를 __명시적으로 직접 작성해야 함__
        - 기존의 exchange()는 deprecated됨 → `exchangeToMono()/Flux()` 로 대체됨
    - 사용 상황
        - 상태 코드별로 다른 처리 로직이 필요한 경우 (예: 200은 데이터, 404는 empty, 500은 error)
        - SSE / NDJSON / Stream처럼 chunk 단위로 처리할 때
```java
Flux<MyDto> flux = webClient.get()
    .uri("/api/items")
    .exchangeToFlux(response -> {
        if (response.statusCode().is2xxSuccessful()) {
            return response.bodyToFlux(MyDto.class);
        } else {
            return response.createException()
                           .flatMapMany(Flux::error);
        }
    });
```
<br/>

### 1-2. 고수준 API

 - __retrieve()__
    - 일반적인 응답 처리용 (고수준 API)
    - 대부분의 REST API 통신에서 사용
    - 특징
        - 내부적으로 `exchangeToMono()` 기반으로 동작하지만, `onStatus()` 체인을 통해 상태 코드별 예외 처리 로직을 자동화
        - 성공 시 `bodyToMono()` 또는 `bodyToFlux()`로 Body를 바로 매핑.
        - 실패 시 `WebClientResponseException` 발생 (Spring 기본 예외).
    - 사용 상황
        - REST API 호출 결과를 단순히 JSON으로 파싱해 DTO로 받는 경우
        - 오류 핸들링이 onStatus()로 충분할 때
```java
Flux<MyDto> flux = webClient.get()
    .uri("/api/items")
    .retrieve()
    .bodyToFlux(MyDto.class);

webClient.get()
    .uri("/api/items")
    .retrieve()
    .onStatus(HttpStatus::is4xxClientError, resp -> Mono.error(new MyBadRequestException()))
    .onStatus(HttpStatus::is5xxServerError, resp -> Mono.error(new MyServerErrorException()))
    .bodyToFlux(MyDto.class);
```
<br/>

 - __bodyToMono() / bodyToFlux()__
    - Body 추출기
    - exchangeToXxx() 또는 retrieve() 이후 응답 Body를 구체적인 타입으로 변환합니다.
    - 특징
        - JSON → POJO 자동 변환 (Jackson 사용)
        - `Mono<T>` : 단일 객체
        - `Flux<T>` : 배열/리스트/스트림 형태 (NDJSON, SSE 등)
    - 사용 상황
        - API가 1개의 결과를 리턴할 때 → `bodyToMono()`
        - API가 여러 개의 데이터 또는 스트림을 리턴할 때 → `bodyToFlux()`
```java
Mono<MyDto> mono = webClient.get()
    .uri("/api/item/1")
    .retrieve()
    .bodyToMono(MyDto.class);

Flux<MyDto> flux = webClient.get()
    .uri("/api/items")
    .retrieve()
    .bodyToFlux(MyDto.class);
```
