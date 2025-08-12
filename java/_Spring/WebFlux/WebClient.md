# WebClient

 - WebClient 기본 설정 및 사용법
 - WebTestClient
 - WebClient 요청 정보 설정(URI 경로 구성, Header, Body 설정)
 - WebClient 응답 처리
 - WebClient 필터
 - WebClient 재시도
 - WebClient 동시 호출
 - 실전 팁 & 키워드


## 1. WebClient 기본

 - `build.gradle`
```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
}
```
<br/>

 - `인스턴스 생성`
```java
// 기본
WebClient client = WebClient.create();

// 기본 URI 매개변수 설정
WebClient client = WebClient.create("http://localhost:8080");

// DefaultWebClientBuilder 클래스 이용
WebClient client = WebClient.builder()
  .baseUrl("http://localhost:8080")
  .defaultCookie("cookieKey", "cookieValue")
  .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE) 
  .defaultUriVariables(Collections.singletonMap("url", "http://localhost:8080"))
  .build();
```
<br/>

 - `타임아웃 설정`
    - ChannelOption.CONNECT_TIMEOUT_MILLIS: 연결 시간 초과 설정
    - ReadTimeoutHandler, WriteTimeoutHandler: 읽기 및 쓰기 시간 제한 설정
    - responseTimeout:응답 시간 초과 설정
    - 클라이언트 요청에 대해서도 타임아웃을 호출할 수 있지만, 이는 HTTP 연결, 읽기/쓰기 또는 응답 타임아웃이 아니라 신호 타임아웃이다.
```java
HttpClient httpClient = HttpClient.create()
  .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
  .responseTimeout(Duration.ofMillis(5000))
  .doOnConnected(conn -> 
    conn.addHandlerLast(new ReadTimeoutHandler(5000, TimeUnit.MILLISECONDS))
      .addHandlerLast(new WriteTimeoutHandler(5000, TimeUnit.MILLISECONDS)));

WebClient client = WebClient.builder()
  .clientConnector(new ReactorClientHttpConnector(httpClient))
  .build();
```
<br/>

 - `요청 준비 - 메서드 정의`
```java
// method를 호출하여 HTTP 메서드 지정
UriSpec<RequestBodySpec> uriSpec = client.method(HttpMethod.POST);

// get, post, delete 같은 단축 메서드 사용
UriSpec<RequestBodySpec> uriSpec = client.post();
```
<br/>

 - `요청 준비 - URL 정의`
```java
RequestBodySpec bodySpec = uriSpec.uri("/resource");

// UriBuilder 함수 사용
RequestBodySpec bodySpec = uriSpec.uri(uriBuilder -> uriBuilder.pathSegment("/resource").build());

// java.net.URL 인스턴스 사용
RequestBodySpec bodySpec = uriSpec.uri(URI.create("/resource"));
```
<br/>

 - `요청 준비 – 본문 정의`
    - 요청 본문, 콘텐츠 유형, 길이, 쿠키 또는 헤더를 설정할 수 있음
```java
RequestHeadersSpec<?> headersSpec = bodySpec.bodyValue("data");
RequestHeadersSpec<?> headersSpec = bodySpec.body(Mono.just(new Foo("name")), Foo.class);
RequestHeadersSpec<?> headersSpec = bodySpec.body(BodyInserters.fromValue("data"));
```
<br/>

 - `요청 준비 – 헤더 정의`
    - 본문을 설정한 후에는 헤더, 쿠키, 그리고 허용되는 미디어 유형을 설정할 수 있음
```java
ResponseSpec responseSpec = headersSpec.header(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
  .accept(MediaType.APPLICATION_JSON, MediaType.APPLICATION_XML)
  .acceptCharset(StandardCharsets.UTF_8)
  .ifNoneMatch("*")
  .ifModifiedSince(ZonedDateTime.now())
  .retrieve();
```
<br/>

 - `응답 받기`
    - 마지막 단계는 요청을 보내고 응답을 받는 것이다.
    - exchangeToMono/exchangeToFlux 또는 retrieve 메서드를 사용
```java
// exchangeToMono 및 exchangeToFlux 메서드를 사용 하면 상태 및 헤더와 함께 ClientResponse 에 액세스할 수 있음
Mono<String> response = headersSpec.exchangeToMono(response -> {
  if (response.statusCode().equals(HttpStatus.OK)) {
      return response.bodyToMono(String.class);
  } else if (response.statusCode().is4xxClientError()) {
      return Mono.just("Error response");
  } else {
      return response.createException()
        .flatMap(Mono::error);
  }
});

// 아래 메서드는 상태 코드가 4xx (클라이언트 오류) 또는 5xx (서버 오류) 인 경우 WebClientException을 throw한다.
Mono<String> response = headersSpec.retrieve()
  .bodyToMono(String.class);
```
<br/>

## 2. WebTestClient

WebTestClient 는 WebFlux 서버 엔드포인트 테스트를 위한 주요 진입점입니다. WebClient 와 매우 유사한 API를 제공하며 , 대부분의 작업을 내부 WebClient 인스턴스에 위임하여 주로 테스트 컨텍스트 제공에 집중합니다.

```java
// 서버 바인딩
// 실행 중인 서버에 대한 실제 요청으로 종단 간 통합 테스트를 완료하려면 bindToServer 메서드를 사용할 수 있습니다.
WebTestClient testClient = WebTestClient
  .bindToServer()
  .baseUrl("http://localhost:8080")
  .build();

// 라우터에 바인딩
// bindToRouterFunction 메서드에 특정 RouterFunction을 전달하여 테스트할 수 있습니다 .
RouterFunction function = RouterFunctions.route(
  RequestPredicates.GET("/resource"),
  request -> ServerResponse.ok().build()
);

WebTestClient
  .bindToRouterFunction(function)
  .build().get().uri("/resource")
  .exchange()
  .expectStatus().isOk()
  .expectBody().isEmpty();

// 웹 핸들러에 바인딩
// WebHandler 인스턴스를 사용하는 bindToWebHandler 메서드를 사용하면 동일한 동작을 얻을 수 있습니다.
WebHandler handler = exchange -> Mono.empty();
WebTestClient.bindToWebHandler(handler).build();

// 애플리케이션 컨텍스트에 바인딩
@Autowired
private ApplicationContext context;

WebTestClient testClient = WebTestClient.bindToApplicationContext(context)
  .build();

// 컨트롤러에 바인딩
@Autowired
private Controller controller;

WebTestClient testClient = WebTestClient.bindToController(controller).build();

// 요청하기
// WebTestClient 객체를 빌드한 후 체인의 모든 후속 작업은 exchange 메서드(응답을 얻는 한 가지 방법) 까지 WebClient 와 유사하다.
// exchange 메서드는 expectStatus, expectBody, expectHeader와 같은 유용한 메서드와 함께 작동하도록 WebTestClient.ResponseSpec 인터페이스를 제공한다.
WebTestClient
  .bindToServer()
    .baseUrl("http://localhost:8080")
    .build()
    .post()
    .uri("/resource")
  .exchange()
    .expectStatus().isCreated()
    .expectHeader().valueEquals("Content-Type", "application/json")
    .expectBody().jsonPath("field").isEqualTo("value");
```

## 3. WebClient 요청 정보 설정(URI 경로 구성, Header, Body 설정)

### 3-1. URI 경로 구성

 - `URI 경로 구성`
```java
// 단순 URI
webClient.get()
  .uri("/products")
  .retrieve()
  .bodyToMono(String.class)
  .onErrorResume(e -> Mono.empty())
  .block();

verifyCalledUrl("/products");

// 세그먼트가 있는 URI
webClient.get()
  .uri(uriBuilder - > uriBuilder
    .path("/products/{id}")
    .build(2))
  .retrieve()
  .bodyToMono(String.class)
  .onErrorResume(e -> Mono.empty())
  .block();

verifyCalledUrl("/products/2");

// 여러 경로 세그먼트가 있는 URI
webClient.get()
  .uri(uriBuilder - > uriBuilder
    .path("/products/{id}/attributes/{attributeId}")
    .build(2, 13))
  .retrieve()
  .bodyToMono(String.class)
  .onErrorResume(e -> Mono.empty())
  .block();

verifyCalledUrl("/products/2/attributes/13");

// URI 쿼리 매개변수 (쿼리 매개변수 즉시 할당)
webClient.get()
  .uri(uriBuilder - > uriBuilder
    .path("/products/")
    .queryParam("name", "AndroidPhone")
    .queryParam("color", "black")
    .queryParam("deliveryDate", "13/04/2019")
    .build())
  .retrieve()
  .bodyToMono(String.class)
  .onErrorResume(e -> Mono.empty())
  .block();

verifyCalledUrl("/products/?name=AndroidPhone&color=black&deliveryDate=13/04/2019");

// URI 쿼리 매개변수 (자리 표시자 이용, "/" 문자가 이스케이프 처리됨)
webClient.get()
  .uri(uriBuilder - > uriBuilder
    .path("/products/")
    .queryParam("name", "{title}")
    .queryParam("color", "{authorId}")
    .queryParam("deliveryDate", "{date}")
    .build("AndroidPhone", "black", "13/04/2019"))
  .retrieve()
  .bodyToMono(String.class)
  .block();

verifyCalledUrl("/products/?name=AndroidPhone&color=black&deliveryDate=13%2F04%2F2019");

// 배열 매개변수 (/products/?tag[]={tag1}&tag[]={tag2})
webClient.get()
  .uri(uriBuilder - > uriBuilder
    .path("/products/")
    .queryParam("tag[]", "Snapdragon", "NFC")
    .build())
  .retrieve()
  .bodyToMono(String.class)
  .onErrorResume(e -> Mono.empty())
  .block();

verifyCalledUrl("/products/?tag%5B%5D=Snapdragon&tag%5B%5D=NFC");

webClient.get()
  .uri(uriBuilder - > uriBuilder
    .path("/products/")
    .queryParam("category", "Phones", "Tablets")
    .build())
  .retrieve()
  .bodyToMono(String.class)
  .onErrorResume(e -> Mono.empty())
  .block();

verifyCalledUrl("/products/?category=Phones&category=Tablets");

webClient.get()
  .uri(uriBuilder - > uriBuilder
    .path("/products/")
    .queryParam("category", String.join(",", "Phones", "Tablets"))
    .build())
  .retrieve()
  .bodyToMono(String.class)
  .onErrorResume(e -> Mono.empty())
  .block();

verifyCalledUrl("/products/?category=Phones,Tablets");
```
<br/>

### 3-2. 요청 Header 설정

 - `WebClient Bean 설정 시 공통 적용`
    - defaultHeader() 사용
    - 특징: 모든 요청에 자동으로 붙음
    - 사용처: API 버전, 공통 인증 토큰 등 항상 필요한 헤더
```java
@Bean
public WebClient webClient(WebClient.Builder builder) {
    return builder
            .baseUrl("https://example.com")
            .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
            .defaultHeader("X-Api-Version", "v1")
            .build();
}
```
<br/>

 - `요청 시 header() 메서드로 개별 지정`
    - 특징: 해당 요청에서만 유효
    - 사용처: 요청마다 다른 값이 필요한 헤더 (예: Authorization Bearer 토큰)
```java
webClient.post()
        .uri("/api/test")
        .header(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
        .header("X-Custom-Header", "ABC123")
        .retrieve()
        .bodyToMono(String.class);
```
<br/>

 - `headers() – 여러 헤더를 한 번에 세팅`
    - 특징: HttpHeaders 객체 조작 가능
    - 사용처: 여러 헤더를 한꺼번에 세팅할 때
```java
webClient.get()
        .uri("/api/data")
        .headers(h -> {
            h.setBearerAuth("jwt-token-value");
            h.setAccept(List.of(MediaType.APPLICATION_JSON, MediaType.APPLICATION_XML));
            h.add("X-Trace-Id", UUID.randomUUID().toString());
        })
        .retrieve()
        .bodyToMono(String.class);
```
<br/>

 - `ExchangeFilterFunction – 공통 전처리 필터`
    - 특징: 모든 요청에 동적으로 헤더 추가 가능
    - 사용처: 요청 추적 ID, 동적 인증 토큰 갱신
```java
@Bean
public WebClient webClient(WebClient.Builder builder) {
    return builder
        .filter((request, next) -> {
            ClientRequest newReq = ClientRequest.from(request)
                    .header("X-Global-Trace-Id", UUID.randomUUID().toString())
                    .build();
            return next.exchange(newReq);
        })
        .build();
}
```
<br/>

### 3-3. 요청 Body 설정

 - `FormData 전송 (application/x-www-form-urlencoded)`
    - BodyInserters.fromFormData() 로 key-value 형태로 데이터 전송
```java
webClient.post()
    .uri("/api/form")
    .contentType(MediaType.APPLICATION_FORM_URLENCODED)
    .body(BodyInserters.fromFormData("username", "testUser")
                        .with("password", "1234"))
    .retrieve()
    .bodyToMono(String.class)
    .block();
```
<br/>

 - `JSON 전송 (application/json)`
    - bodyValue()는 Jackson이 자동으로 객체 → JSON 변환
```java
UserRequest request = new UserRequest("홍길동", 30);

webClient.post()
    .uri("/api/json")
    .contentType(MediaType.APPLICATION_JSON)
    .bodyValue(request) // 객체를 바로 JSON 변환
    .retrieve()
    .bodyToMono(String.class)
    .block();
```
<br/>

 - `Multipart 전송 (multipart/form-data)`
    - BodyInserters.fromMultipartData() 를 사용하면 파일과 일반 필드를 함께 전송 가능
```java
File file = filePath.toFile();

webClient.post()
    .uri("/api/upload")
    .contentType(MediaType.MULTIPART_FORM_DATA)
    .body(BodyInserters.fromMultipartData("file", new FileSystemResource(file))
                        .with("description", "샘플 파일"))
    .retrieve()
    .bodyToMono(String.class)
    .block();
```
<br/>

## 4. WebClient 응답 처리

 - `기본 응답 처리 (bodyToMono)`
    - 특징: 간단한 타입 매핑
    - 사용처: 응답 JSON 구조가 단순하거나 DTO로 바로 매핑할 수 있을 때
```java
Mono<String> responseMono = webClient.get()
        .uri("/api/hello")
        .retrieve()
        .bodyToMono(String.class);  // JSON → String 변환

String result = responseMono.block(); // 동기 처리
```
<br/>

 - `제네릭 타입 응답 (ParameterizedTypeReference)`
    - 특징: `List<T>`나 `ApiResponse<T>`처럼 제네릭 포함된 구조를 매핑할 때 필요.
    - 사용처: 공통 응답 래퍼(code, message, data) 구조가 있는 경우.
```java
Mono<ApiResponse<UserDto>> mono = webClient.get()
        .uri("/api/user/1")
        .retrieve()
        .bodyToMono(new ParameterizedTypeReference<ApiResponse<UserDto>>() {});
```
<br/>

 - `상태 코드별 처리 (onStatus)`
    - 특징: HTTP 상태 코드에 따라 다른 예외 발생 가능
    - 사용처: API 응답 코드에 맞춰 예외 처리 로직이 필요한 경우
```java
Mono<UserDto> mono = webClient.get()
        .uri("/api/user/1")
        .retrieve()
        .onStatus(HttpStatusCode::is4xxClientError, resp ->
            resp.bodyToMono(String.class)
                .flatMap(body -> Mono.error(new IllegalArgumentException("클라이언트 오류: " + body)))
        )
        .onStatus(HttpStatusCode::is5xxServerError, resp ->
            resp.bodyToMono(String.class)
                .flatMap(body -> Mono.error(new IllegalStateException("서버 오류: " + body)))
        )
        .bodyToMono(UserDto.class);
```
<br/>

 - `응답 전체 직접 처리 (exchangeToMono)`
    - 특징: ClientResponse 전체에 접근 가능 (헤더, 상태코드, 바디)
    - 사용처: retrieve()보다 세밀하게 상태·헤더·바디를 모두 다뤄야 할 때
```java
Mono<UserDto> mono = webClient.get()
        .uri("/api/user/1")
        .exchangeToMono(clientResponse -> {
            if (clientResponse.statusCode().is2xxSuccessful()) {
                return clientResponse.bodyToMono(UserDto.class);
            } else {
                return clientResponse.bodyToMono(String.class)
                        .flatMap(body -> Mono.error(new RuntimeException("오류 응답: " + body)));
            }
        });
```
<br/>

 - `응답 바디 없이 상태 코드만 확인 (toBodilessEntity)`
    - 특징: 바디가 필요 없는 경우 (204 No Content 같은 응답)
```java
Mono<ResponseEntity<Void>> mono = webClient.delete()
        .uri("/api/user/1")
        .retrieve()
        .toBodilessEntity();

ResponseEntity<Void> entity = mono.block();
System.out.println("상태 코드: " + entity.getStatusCode());
```
<br/>

 - `ResponseEntity<T>로 전체 응답 받기`
```java
Mono<ResponseEntity<UserDto>> mono = webClient.get()
        .uri("/api/user/1")
        .retrieve()
        .toEntity(UserDto.class);

ResponseEntity<UserDto> response = mono.block();
System.out.println("상태 코드: " + response.getStatusCode());
System.out.println("헤더: " + response.getHeaders());
System.out.println("바디: " + response.getBody());
```
<br/>

## 5. WebClient 필터

필터는 클라이언트 요청(또는 응답)을 가로채고, 검사하고, 수정할 수 있다.

 - 필터는 로직이 한곳에 집중되어 있기 때문에 모든 요청에 기능을 추가하는 데 매우 적합하다.
 - 사용 사례: 클라이언트 요청 모니터링, 수정, 로깅, 인증

Spring Reactive에서 필터는 함수형 인터페이스 ExchangeFilterFunction 의 인스턴스로 필터 함수는 두 개의 매개변수를 가진다.

```java
@FunctionalInterface
public interface ExchangeFilterFunction {
    Mono<ClientResponse> filter(ClientRequest request, ExchangeFunction next);

    default ExchangeFilterFunction andThen(ExchangeFilterFunction afterFilter) {
        Assert.notNull(afterFilter, "ExchangeFilterFunction must not be null");
        return (request, next) -> {
            return this.filter(request, (afterRequest) -> {
                return afterFilter.filter(afterRequest, next);
            });
        };
    }

    default ExchangeFunction apply(ExchangeFunction exchange) {
        Assert.notNull(exchange, "ExchangeFunction must not be null");
        return (request) -> {
            return this.filter(request, exchange);
        };
    }

    static ExchangeFilterFunction ofRequestProcessor(Function<ClientRequest, Mono<ClientRequest>> processor) {
        Assert.notNull(processor, "ClientRequest Function must not be null");
        return (request, next) -> {
            Mono var10000 = (Mono)processor.apply(request);
            Objects.requireNonNull(next);
            return var10000.flatMap(next::exchange);
        };
    }

    static ExchangeFilterFunction ofResponseProcessor(Function<ClientResponse, Mono<ClientResponse>> processor) {
        Assert.notNull(processor, "ClientResponse Function must not be null");
        return (request, next) -> {
            return next.exchange(request).flatMap(processor);
        };
    }
}
```
<br/>

### 5-1. 사용자 정의 필터

 - `요청 필터 기본 사용 예제`
```java
// 요청 필터: GET 요청에 대해 전역 카운터 증가
ExchangeFilterFunction countingFunction = (clientRequest, nextFilter) -> {
    HttpMethod httpMethod = clientRequest.method();
    if (httpMethod == HttpMethod.GET) {
        getCounter.incrementAndGet();
    }
    return nextFilter.exchange(clientRequest);
};

// 요청 필터: URL 경로에 버전 번호 추가
// 새 요청 객체를 생성하고 수정된 URL로 설정
ExchangeFilterFunction urlModifyingFilter = (clientRequest, nextFilter) -> {
    String oldUrl = clientRequest.url().toString();
    URI newUrl = URI.create(oldUrl + "/" + version);
    ClientRequest filteredRequest = ClientRequest.from(clientRequest)
      .url(newUrl)
      .build();
    return nextFilter.exchange(filteredRequest);
};

// 요청 필터: 요청 메서드와 URL 로깅
ExchangeFilterFunction loggingFilter = (clientRequest, nextFilter) -> {
    printStream.print("Sending request " + clientRequest.method() + " " + clientRequest.url());
    return nextFilter.exchange(clientRequest);
};
```
<br/>

### 5-2. 필터 응용 예제

 - `WebClientConfig`
```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient(WebClient.Builder builder, TokenManager tokenManager) {
        return builder
                .baseUrl("https://api.example.com")
                .filter(CorrelationIdFilter.correlationId())     // 공통 상관관계ID
                .filter(LoggingFilter.compact())                  // 요청/응답 로깅 (바디 소량)
                .filter(UnauthorizedRetryFilter.refreshOnce(tokenManager)) // 401 → 토큰 리프레시 후 1회 재시도
                .build();
    }
}
```
<br/>

 - `상관관계 ID 자동 주입 필터 (헤더 추가)`
    - 모든 요청에 추적용 ID가 필요할 때(서버 로그 상관관계 추적, 분산 트레이싱 등)
```java
import org.springframework.web.reactive.function.client.*;
import java.util.Optional;
import java.util.UUID;

public final class CorrelationIdFilter {
    private static final String HEADER = "X-Correlation-Id";

    private CorrelationIdFilter() {}

    public static ExchangeFilterFunction correlationId() {
        return (request, next) -> {
            String cid = Optional.ofNullable(request.headers().getFirst(HEADER))
                    .orElse(UUID.randomUUID().toString());

            ClientRequest newReq = ClientRequest.from(request)
                    .header(HEADER, cid)
                    .build();

            return next.exchange(newReq);
        };
    }
}
```
<br/>

 - `요청/응답 로깅 필터 (작은 바디 버퍼링)`
    - 응답 바디는 한 번만 소비 가능합니다. 위처럼 버퍼링 후 ClientResponse를 재생성해야 이후 단계에서 정상 파싱됩니다.
    - 대용량/스트리밍은 바디 로깅을 건너뛰어야 한다.
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.MediaType;
import org.springframework.web.reactive.function.client.*;
import reactor.core.publisher.Mono;

public final class LoggingFilter {
    private static final Logger log = LoggerFactory.getLogger(LoggingFilter.class);
    private static final int MAX_BODY_LOG = 2000; // 너무 길면 잘라서 로깅

    private LoggingFilter() {}

    public static ExchangeFilterFunction compact() {
        return (request, next) -> {
            long start = System.currentTimeMillis();

            log.info("--> {} {} {}", request.method(), request.url(),
                    redactAuth(request.headers().asHttpHeaders().toString()));

            return next.exchange(request)
                    .flatMap(response -> {
                        long took = System.currentTimeMillis() - start;
                        var status = response.statusCode();
                        var mt = response.headers().contentType().orElse(null);

                        // 텍스트/JSON 계열만 바디 로깅
                        if (isTextLike(mt)) {
                            return response.bodyToMono(String.class)
                                    .defaultIfEmpty("")
                                    .map(body -> {
                                        log.info("<-- {} {} {} ({} ms)\n{}",
                                                request.method(), request.url(), status.value(), took,
                                                truncate(body, MAX_BODY_LOG));

                                        // 바디를 한 번 소비했으니, 새 response로 복원
                                        return ClientResponse.create(status, response.strategies())
                                                .headers(h -> h.addAll(response.headers().asHttpHeaders()))
                                                .cookies(c -> c.addAll(response.cookies()))
                                                .body(body)
                                                .build();
                                    });
                        } else {
                            log.info("<-- {} {} {} ({} ms)", request.method(), request.url(), status.value(), took);
                            return Mono.just(response);
                        }
                    });
        };
    }

    private static boolean isTextLike(MediaType mt) {
        if (mt == null) return false;
        return mt.includes(MediaType.APPLICATION_JSON)
                || mt.includes(MediaType.APPLICATION_XML)
                || mt.getType().equalsIgnoreCase("text");
    }

    private static String truncate(String s, int max) {
        return (s.length() <= max) ? s : s.substring(0, max) + "...(truncated)";
    }

    private static String redactAuth(String headers) {
        // 필요 시 Authorization 값 마스킹
        return headers.replaceAll("(?i)(Authorization:\\s*Bearer\\s+)[^\\n]+", "$1******");
    }
}
```
<br/>

 - `401 Unauthorized 시 토큰 갱신 후 1회 재시도 필터`
    - 무한 루프 방지를 위해 딱 1회만 재시도
    - TokenManager는 내부적으로 Refresh Token을 이용해 새 액세스 토큰을 받아 저장 후 반환하도록 구현
```java
import org.springframework.http.HttpStatus;
import org.springframework.web.reactive.function.client.*;
import reactor.core.publisher.Mono;

public final class UnauthorizedRetryFilter {

    private UnauthorizedRetryFilter() {}

    public static ExchangeFilterFunction refreshOnce(TokenManager tm) {
        return (request, next) ->
                // 1차 요청
                next.exchange(attachBearer(request, tm.get()))
                        .flatMap(resp -> {
                            if (resp.statusCode() == HttpStatus.UNAUTHORIZED) {
                                // 토큰 갱신 후 1회 재시도
                                return tm.refreshAndGet()
                                        .flatMap(newToken ->
                                                next.exchange(attachBearer(request, newToken)));
                            }
                            return Mono.just(resp);
                        });
    }

    private static ClientRequest attachBearer(ClientRequest req, String token) {
        return ClientRequest.from(req)
                .headers(h -> h.setBearerAuth(token))
                .build();
    }
}

public interface TokenManager {
    String get();                // 현재 액세스 토큰 동기 조회
    Mono<String> refreshAndGet(); // 토큰 갱신 + 최신 토큰 반환
}
```
<br/>

## 6. WebClient 재시도(Retry)

Spring WebClient에서 재시도(Retry)를 구현하는 방법으로는 Reactor의 retry(), retryWhen()을 이용하거나, Resilience4j 같은 외부 라이브러리를 붙이는 방법이 존재하낟.

 - `단순 재시도(retry)`
    - 모든 예외에 대해 재시도하므로, HTTP 4xx 같은 경우에도 재시도할 수 있음
```java
webClient.get()
    .uri("/api/data")
    .retrieve()
    .bodyToMono(String.class)
    .retry(3) // 실패 시 최대 3번 재시도
    .block();
```
<br/>

 - `조건부 재시도 (retryWhen)`
    - retryWhen()을 사용하면 재시도 조건과 대기 시간(backoff)를 지정할 수 있다.
    - fixedDelay → 고정 대기 시간
    - backoff → 점점 증가하는 대기 시간 (지수 백오프)
    - filter()로 재시도 대상 예외 지정 가능
```java
import reactor.util.retry.Retry;
import java.time.Duration;

webClient.get()
    .uri("/api/data")
    .retrieve()
    .onStatus(HttpStatusCode::is4xxClientError, clientResponse -> {
        // 4xx이면 재시도 없이 바로 에러
        return Mono.error(new IllegalStateException("Client error: " + clientResponse.statusCode()));
    })
    .bodyToMono(String.class)
    .retryWhen(
        Retry.fixedDelay(3, Duration.ofSeconds(2)) // 2초 간격으로 3번 재시도
            .filter(throwable -> !(throwable instanceof IllegalStateException)) // 특정 예외 제외
            .onRetryExhaustedThrow((retryBackoffSpec, retrySignal) -> retrySignal.failure())
    )
    .block();
```
<br/>

 - `공통 재시도 로직 적용 (ExchangeFilterFunction)`
```java
@Bean
public WebClient webClient(WebClient.Builder builder) {
    return builder
        .baseUrl("https://example.com")
        .filter(retryFilter())
        .build();
}

private ExchangeFilterFunction retryFilter() {
    return (request, next) -> next.exchange(request)
        .flatMap(response -> {
            if (response.statusCode().is5xxServerError()) {
                return Mono.error(new RuntimeException("Server error"));
            }
            return Mono.just(response);
        })
        .retryWhen(
            Retry.backoff(3, Duration.ofMillis(500))
                 .maxBackoff(Duration.ofSeconds(5))
                 .jitter(0.5) // 랜덤성 부여
        );
}
```
<br/>

 - `외부 라이브러리 (Resilience4j)`
    - Resilience4j는 재시도 + 회로차단 + RateLimiter까지 가능해서, 프로덕션에서 안정성을 높이는 데 자주 사용된다.
    - 예외 타입, 재시도 간격, 최대 횟수 세밀 제어 가능
    - CircuitBreaker 등과 조합 가능
```java
import io.github.resilience4j.retry.Retry;
import io.github.resilience4j.retry.RetryConfig;
import java.time.Duration;

RetryConfig config = RetryConfig.custom()
    .maxAttempts(3)
    .waitDuration(Duration.ofSeconds(1))
    .retryExceptions(WebClientRequestException.class)
    .build();

Retry retry = Retry.of("myRetry", config);

Mono<String> result = Mono.fromCallable(() ->
        webClient.get()
            .uri("/api/data")
            .retrieve()
            .bodyToMono(String.class)
            .block()
    )
    .transformDeferred(RetryOperator.of(retry));
```
<br/>

## 7. WebClient 동시 호출

 - `동시 호출`
    - 정적 fromIterable 메서드를 사용하여 userId 목록에서 Flux를 생성하고, flatMap을 호출
    - 연산이 병렬로 진행되기 때문에 결과 순서를 알 수 없음
    - 입력 순서를 유지해야 하는 경우 flatMapSequential 연산자를 사용
```java
WebClient webClient = WebClient.create("http://localhost:8080");

public Mono<User> getUser(int id) {
    LOG.info(String.format("Calling getUser(%d)", id));

    return webClient.get()
        .uri("/user/{id}", id)
        .retrieve()
        .bodyToMono(User.class);
}

// 동시 호출
public Flux fetchUsers(List userIds) {
    return Flux.fromIterable(userIds)
        .flatMap(this::getUser);
}
```
<br/>

 - `동일한 유형을 반환하는 여러 서비스 동시 호출`
    - 정적 메서드 merge를 사용
    - merge를 사용하면 두 개 이상의 Flux를 하나의 결과로 결합
```java
public Mono<User> getUser(int id) {
    LOG.info(String.format("Calling getUser(%d)", id));

    return webClient.get()
        .uri("/user/{id}", id)
        .retrieve()
        .bodyToMono(User.class);
}

public Mono<User> getOtherUser(int id) {
    return webClient.get()
        .uri("/otheruser/{id}", id)
        .retrieve()
        .bodyToMono(User.class);
}

public Flux fetchUserAndOtherUser(int id) {
    return Flux.merge(getUser(id), getOtherUser(id));
}
```
<br/>

 - `다양한 유형을 반환하는 여러 서비스 동시 호출`
    - 두 서비스가 동일한 결과를 반환할 확률은 매우 낮습니다.
    - 일반적으로는 다른 유형의 응답을 제공하는 또 다른 서비스가 있을 것이며, 우리의 목표는 두 개(또는 그 이상)의 응답을 병합하는 것이다.
```java
public Mono fetchUserAndItem(int userId, int itemId) {
    Mono user = getUser(userId);
    Mono item = getItem(itemId);

    return Mono.zip(user, item, UserWithItem::new);
}
```
<br/>

## 8. 실전 팁 & 키워드

 - 전역 공통 헤더(버전, App-Id 등)는 defaultHeader보다 필터 주입이 더 유연합니다(동적 값 주입 용이).
 - 로깅 필터는 운영에서는 최소화/비활성, 개발/스테이징에서 상세 출력 권장.
 - 429/5xx 재시도는 필터보단 Reactor 연산자(retryWhen + backoff) 쪽이 관리하기 편함.


<br/>

### 8-1. 실전 팁

 - `커넥션/타임아웃/버퍼 사이즈 튜닝`
    - 큰 JSON이면 maxInMemorySize 꼭 올리기
    - 짧은 타임아웃 + 재시도(백오프) 조합이 운영에서 안전
```java
@Bean
WebClient webClient() {
    var provider = ConnectionProvider.builder("http")
            .maxConnections(200)                // 동시 커넥션
            .pendingAcquireMaxCount(1000)       // 큐 대기
            .maxIdleTime(Duration.ofSeconds(30))
            .maxLifeTime(Duration.ofMinutes(5))
            .build();

    var http = HttpClient.create(provider)
            .compress(true)                     // gzip/deflate
            .followRedirect(true)
            .responseTimeout(Duration.ofSeconds(5))
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
            .doOnConnected(c -> c
                .addHandlerLast(new ReadTimeoutHandler(5))
                .addHandlerLast(new WriteTimeoutHandler(5)));

    return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(http))
            .codecs(c -> c.defaultCodecs().maxInMemorySize(16 * 1024 * 1024)) // 16MB
            .build();
}
```
<br/>

 - `재시도(HTTP/비즈니스 조건 + 지수백오프 + 지터)`
    - 4xx라도 429는 재시도 후보. Retry-After 헤더가 있으면 읽어 반영하면 더 좋음
```java
webClient.get().uri("/v1/items")
    .retrieve()
    .onStatus(s -> s.value()==429 || s.is5xxServerError(), r -> r.createException()) // 재시도 대상
    .bodyToMono(ItemDto.class)
    .retryWhen(Retry.backoff(3, Duration.ofMillis(200))
        .maxBackoff(Duration.ofSeconds(3))
        .jitter(0.5)
        .filter(ex -> ex instanceof WebClientResponseException)  // 커스텀 필터
    )
    .timeout(Duration.ofSeconds(3));
```
<br/>

 - `로깅 (요청/응답 헤더 마스킹 + 바디 크기 제한 + 텍스트만)`
    - Authorization/쿠키/PII 마스킹
    - 스트리밍/바이너리는 바디 로깅 금지
    - 바디 로깅 시 한 번만 소비 가능 → 버퍼링 후 ClientResponse 재생성

<br/>

 - `컨텍스트(추적ID, 사용자ID) 전파`
    - MDC를 쓰면 스레드 hop에서 깨지니 Reactor Context → MDC 브리지(logback Mapped Diagnostic Context 연동 또는 전용 라이브러리) 고려.
```java
return webClient.get().uri("/v1/x")
    .contextWrite(ctx -> ctx.put("cid", currentCorrelationId()))
    .retrieve().bodyToMono(String.class)
    .doOnEach(sig -> {
        String cid = sig.getContextView().getOrDefault("cid", "n/a");
        log.info("cid={} event={}", cid, sig.getType());
    });
```
<br/>

 - `큰 파일 업로드/다운로드는 스트리밍`
    - 메모리 폭주 방지. Backpressure 고려. 필요 시 limitRate()
```java
// 다운로드 스트리밍
Flux<DataBuffer> body = webClient.get().uri("/big.bin")
    .retrieve().bodyToFlux(DataBuffer.class);
DataBufferUtils.write(body, destPath, CREATE).block();

// 업로드 스트리밍 (파일→DataBuffer Flux)
Flux<DataBuffer> file = DataBufferUtils.read(resource, new DefaultDataBufferFactory(), 64 * 1024);
webClient.post().uri("/upload")
    .contentType(MediaType.APPLICATION_OCTET_STREAM)
    .body(file, DataBuffer.class)
    .retrieve().toBodilessEntity().block();
```
<br/>

 - `에러 바디를 예외에 보존`
    - onStatus에서 response.bodyToMono(String.class)를 읽어 커스텀 예외에 넣어 로깅/알람에 첨부

 - `헤더 기반 분기/캐싱/조건부 요청`
    - ETag/If-None-Match, If-Modified-Since로 트래픽 절감
    - 클라이언트 캐시 필터(ETag 저장/재사용) 구현하면 비용 큰 API에 효과 큼
 - `HTTP/2 활성화`
    - 서버가 HTTP/2면 Reactor Netty로 자동 협상(ALPN). 멀티플렉싱으로 커넥션 덜 쓰고 레이턴시 ↓
    - LB/프록시가 H2를 끝단에서 끊지 않는지 확인
 - `테스트 전략`
    - 통합: WireMock(녹화/리플레이, 지연/에러 시뮬), MockWebServer(OkHttp), Spring WebTestClient
    - 리액티브: StepVerifier + VirtualTimeScheduler로 타임아웃/재시도 시나리오 빠르게 검증

<br/>

### 8-2. 도입하면 좋은 키워드

 - `Resilience4j (CircuitBreaker, Retry, RateLimiter, Bulkhead, TimeLimiter)`
    - 호출 폭주/장애 전파 차단. WebClient에 데코레이터 형태로 감싸 적용이 쉽고, Micrometer와 잘 붙음.
 - `Micrometer + Prometheus/Grafana`
    - reactor.netty.http.client.*/http.client.requests 메트릭 자동 수집(지연, 성공률, 타임아웃, 커넥션 풀 상태).
    - 슬로우콜 히트맵으로 임계치 찾기.
 - `Micrometer Tracing(OpenTelemetry/Brave)`
    - traceparent/baggage 전파. 분산 트레이싱(Zipkin/Jaeger/OTel Collector)으로 원인 추적.
 - `Backoff with Jitter, Hedging Requests`
    - 지수 백오프 + 지터로 “집단 동조(thundering herd)” 완화.
    - 핵심 읽기 API는 헤징(중복소수요청) 도입 시 tail latency 큰 폭 개선(주의: idempotent일 때만).
 - `Idempotency-Key`
    - POST도 멱등화. 재시도 시 중복 실행 방지(결제/주문 등).
 - `ConnectionProvider 튜닝`
    - maxConnections, pendingAcquireMaxCount, maxIdleTime, maxLifeTime 설정으로 커넥션 고갈/고아방지.
 - `Request Collapsing / Coalescing`
    - 같은 키/짧은 시간 동일 요청 합치기(캐시 or in-flight dedup). 외부 트래픽/비용 절감.
 - `Retry-After 준수`
    - 429/503의 Retry-After 읽어 대기시간 반영. API 제공자 매너.
 - `Configuration per-route`
    - 엔드포인트 중요도별로 타임아웃/재시도/풀을 달리. (예: 결제=짧은 타임아웃, 낮은 재시도)
 - `MDC/PII Redaction`
    - 로그 보안 기본기. 마스킹/슬로건 규칙을 중앙 필터화.
 - `HTTP Client-side Cache`
    - ETag/Cache-Control 기반 라이트 캐시. Caffeine 등과 조합.
 - `Fail-fast & Fallback`
    - 핵심 의존 실패 시 빠르게 실패하고, 대체 경로(캐시·기본값·축약 응답) 제공.

<br/>

## 부록

### 1. 개인적인 사용 팁

 - 도메인별로 WebClient 빈 등록
    - 타임아웃 설정
    - 필터 설정(ExchangeFilterFunction)
    - 공통 요청 설정(BaseURL, 공통 헤더)
 - 편의 메서드
    - WebClient를 실행하는 편의 메서드를 작성
        - XxxClientRequest (headers, url, queryParams, body, method)를 매개변수로 받음
        - Mono - XxxClientResponse (code, body)를 응답
        - 200, 4xx, 5xx 등 모두 XxxClientResponse를 반환
```java
HttpClient httpClient = HttpClient.create()
    .compress(true)
    .responseTimeout(Duration.ofSeconds(2))       // 응답 대기
    .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 1500)
    .doOnConnected(conn -> conn
        .addHandlerLast(new ReadTimeoutHandler(2))
        .addHandlerLast(new WriteTimeoutHandler(2)));

WebClient client = WebClient.builder()
    .baseUrl("https://api.example.com")
    .clientConnector(new ReactorClientHttpConnector(httpClient))
    .defaultHeaders(h -> {
        h.set(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE);
        // 도메인 공통 헤더
    })
    .codecs(c -> c.defaultCodecs().maxInMemorySize(512 * 1024)) // 512KB
    .filter(loggingFilter())     // 마스킹 포함
    .filter(meteringFilter())    // 메트릭/트레이싱
    .build();

public Mono<XxxClientResponse> executeAsync(XxxClientRequest request) {
    return webClient.method(request.getMethod())
            .uri(request.getUrl())
            .headers(headers -> {
                if (request.headers() != null) h.addAll(request.headers());
            })
            .bodyValue(StringUtils.hasText(request.getBody()) ? request.getBody() : "")
            .retrieve()
            .bodyToMono(String.class)
            .map(XxxClientResponse::of)
            .onErrorResume(this::handleError);
}
```
