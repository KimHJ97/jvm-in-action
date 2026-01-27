# WebClient 치트 시트

## 1. API

### 1-1. 요청 단계 체이닝(prepare/build)

 - __요청 만들기__
    - `webClient.get()/post()/put()/delete()/patch()/head()/options()`
    - `uri(String, Object...)`, `uri(URI)`, `uri(Function<UriBuilder, URI>)`
    - `attribute(String, Object)`, `attributes(Consumer<Map<String,Object>>)`
        - → 필터/코덱에서 꺼내 쓰는 per-request 메타데이터 전달용
 - __헤더/쿠키__
    - `accept(MediaType...)`
    - `contentType(MediaType)`
    - `headers(Consumer<HttpHeaders>)` / `header(String, String...)`
    - `cookie(String, String)` / `cookies(Consumer<MultiValueMap<String, String>>)`
    - (빌더에서 공통 설정) `defaultHeader(...)`, `defaultCookie(...)`, `defaultRequest(...)`
 - __바디__
    - `bodyValue(Object)` : 단건 POJO/문자열/바이트 등
    - `body(BodyInserter<?, ? super ClientHttpRequest>)` : 폼/파일 업로드/멀티파트 등 커스텀
    - `body(Publisher<T,?>, Class<T>)` : Flux/Mono 등 리액티브 스트림 바디
    - `contentLength(long)` : 필요 시 명시

<br/>

### 1-2. 전송/응답 단계 체이닝(send/handle)

 - __고수준(REST 기본용)__
    - `retrieve()`
        - `onStatus(Predicate<HttpStatusCode>`, `Function<ClientResponse,Mono<? extends Throwable>>)`
        - `bodyToMono(Foo.class)`, `bodyToFlux(Foo.class)`
        - `toEntity(Foo.class)`, `toEntityList(Foo.class)`, `toEntityFlux(Foo.class)`
        - `toBodilessEntity()` (HEAD/204 등 바디 없는 응답)
 - __저수준(세밀 제어/스트리밍)__
    - `exchangeToMono(Function<ClientResponse, Mono<T>>)`
    - `exchangeToFlux(Function<ClientResponse, Flux<T>>)`
    - `ClientResponse로 상태/헤더/바디를 직접 분기`
    - `response.bodyToMono/Flux(...)`, `response.createException()` 등

<br/>

### 1-3. 기타 유용한 체이닝/빌더 기능

 - `mutate()` : 기존 WebClient에서 일부만 바꾼 새 인스턴스 생성
 - `filter(ExchangeFilterFunction)` : 로깅/공통 헤더/리트라이/메트릭 등
 - `clientConnector(...)` : Reactor Netty 등 커넥터 교체
 - `codecs(configurer -> ...)` : 코덱 버퍼/메모리 제한, maxInMemorySize 등
 - `baseUrl(String)` : 모든 상대 uri의 기준 URL

<br/>

## 2. WebClient 설정

WebClient 설정(API 레벨에서 다루는 빌더 설정)에는 기본 설정(공통 값, 타임아웃, 코덱 등) 과 고급 설정(필터 체인, 커넥터, ConnectionProvider, SSL, 로깅 등) 이 있다.

 - __주요 설정__
    - __기본 설정__: `baseUrl`, `defaultHeader`, `defaultCookie` (모든 요청 공통 값)
    - __연결 설정__: `clientConnector`, `ConnectionProvider`, `HttpClient` (커넥션 풀, 타임아웃, SSL)
    - __코덱 설정__: `exchangeStrategies` (Jackson, ByteBuffer, Memory 제한)
    - __필터 설정__: `filter()` (인증, 로깅, 리트라이 등)
    - __로깅/디버그__: `wiretap()`, `logging.level` (요청/응답 패킷 보기)
 - __주요 설정 구분__
    - __성능 최적화__: ConnectionProvider (maxConnections, idleTime)
    - __안정성__: Timeout (connect, read, write, response)
    - __편의성__: baseUrl, defaultHeader, defaultCookie
    - __확장성__: ExchangeFilterFunction (로깅/토큰 주입 등)
    - __보안__: SSLContext, 인증서 검증
    - __디버깅__: wiretap(), logging.level TRACE

<br/>

 - __설정 예시__
    - `baseUrl()`: 기본 도메인 지정 (상대경로 요청 시 자동 붙음)
    - `defaultHeader()`: 모든 요청에 공통 헤더 추가
    - `defaultCookie()`: 모든 요청에 공통 쿠키 추가
    - `defaultRequest()`: 공통 요청 메타데이터나 헤더 추가용
    - `exchangeStrategies()`: Jackson/ObjectMapper, Codec, Buffer 제한 등 설정
    - `clientConnector()`: Netty 커넥터 설정 (Connection Pool, Timeout 등)
    - `filter()`: 요청/응답 전후 처리 (로깅, 토큰 주입 등)
```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient(ObjectMapper objectMapper) {
        ConnectionProvider provider = ConnectionProvider.builder("custom")
                .maxConnections(200)
                .maxIdleTime(Duration.ofSeconds(30))
                .maxLifeTime(Duration.ofMinutes(5))
                .pendingAcquireTimeout(Duration.ofSeconds(10))
                .evictInBackground(Duration.ofSeconds(60))
                .build();

        HttpClient httpClient = HttpClient.create(provider)
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
                .responseTimeout(Duration.ofSeconds(5))
                .doOnConnected(conn ->
                        conn.addHandlerLast(new ReadTimeoutHandler(10))
                            .addHandlerLast(new WriteTimeoutHandler(10)))
                .compress(true)
                .wiretap("reactor.netty.http.client", LogLevel.DEBUG, AdvancedByteBufFormat.TEXTUAL);

        ExchangeStrategies strategies = ExchangeStrategies.builder()
                .codecs(config -> {
                    config.defaultCodecs().maxInMemorySize(16 * 1024 * 1024);
                    config.defaultCodecs().jackson2JsonEncoder(
                            new Jackson2JsonEncoder(objectMapper, MediaType.APPLICATION_JSON));
                    config.defaultCodecs().jackson2JsonDecoder(
                            new Jackson2JsonDecoder(objectMapper, MediaType.APPLICATION_JSON));
                })
                .build();

        return WebClient.builder()
                .baseUrl("https://api.domain.com")
                .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
                .clientConnector(new ReactorClientHttpConnector(httpClient))
                .exchangeStrategies(strategies)
                .filter(logRequest())
                .filter(logResponse())
                .build();
    }

    private ExchangeFilterFunction logRequest() {
        return (req, next) -> {
            log.info("[REQ] {} {}", req.method(), req.url());
            return next.exchange(req);
        };
    }

    private ExchangeFilterFunction logResponse() {
        return ExchangeFilterFunction.ofResponseProcessor(res -> {
            log.info("[RES] {} {}", res.statusCode(), res.headers().asHttpHeaders());
            return Mono.just(res);
        });
    }
}
```
<br/>

### 2-1. Reactor Netty 기반 고급 설정 (HttpClient, ConnectionProvider)

 - `maxConnections`: 동시에 유지할 커넥션 최대 수 (DB 커넥션 풀과 비슷)
 - `maxIdleTime`: 유휴 상태로 얼마나 둘지
 - `maxLifeTime`: 커넥션의 최대 수명 (오래된 커넥션 정리용)
 - `pendingAcquireTimeout`: 풀에서 커넥션을 기다리는 최대 시간
 - `responseTimeout`: 응답을 기다리는 최대 시간
 - `ReadTimeoutHandler`/ `WriteTimeoutHandler`: 데이터 송수신 타임아웃
```java
ConnectionProvider provider = ConnectionProvider.builder("custom")
        .maxConnections(100)                // 커넥션 풀 최대 개수
        .maxIdleTime(Duration.ofSeconds(20))// Idle 커넥션 유지 시간
        .maxLifeTime(Duration.ofSeconds(60))// 커넥션 수명
        .pendingAcquireTimeout(Duration.ofSeconds(60)) // 커넥션 요청 대기 시간
        .evictInBackground(Duration.ofSeconds(120))    // 백그라운드 커넥션 정리 주기
        .build();

HttpClient httpClient = HttpClient.create(provider)
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000) // TCP 연결 맺기 타임아웃. 서버로 연결 시도부터, 3-Way Handshake 완료까지 시간
        .responseTimeout(Duration.ofSeconds(5)) // 응답 수신 전체에 대한 타임아웃. 요청 후 전체 응답이 안오면 연결 종료
        .doOnConnected(conn -> // TCP 연결 후 채널 핸들러
                conn.addHandlerLast(new ReadTimeoutHandler(10))     // 읽기 타임아웃
                    .addHandlerLast(new WriteTimeoutHandler(10)));  // 쓰기 타임아웃

WebClient webClient = WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .build();
```
<br/>

### 2-2. Exchange Strategies (코덱 / 메모리 제한 / Jackson 설정)

 - `maxInMemorySize()`: 응답 body를 버퍼링할 수 있는 최대 크기 (기본 256KB → 16MB 권장)
 - Jackson 커스터마이징 시 `objectMapper` 직접 등록 가능
```java
ExchangeStrategies strategies = ExchangeStrategies.builder()
    .codecs(configurer -> {
        configurer.defaultCodecs().maxInMemorySize(16 * 1024 * 1024); // 16MB
        configurer.defaultCodecs().jackson2JsonEncoder(new Jackson2JsonEncoder(objectMapper, MediaType.APPLICATION_JSON));
        configurer.defaultCodecs().jackson2JsonDecoder(new Jackson2JsonDecoder(objectMapper, MediaType.APPLICATION_JSON));
    })
    .build();

WebClient client = WebClient.builder()
    .exchangeStrategies(strategies)
    .build();
```
<br/>

### 2-3. 필터 (ExchangeFilterFunction)

 - 대표적인 필터 활용
    - 로깅: 요청/응답 로그 기록
    - 인증: Authorization 헤더 자동 추가
    - 리트라이: 5xx 응답 시 재시도
    - 메트릭: Prometheus/Zipkin 추적 데이터
    - 캐시: ETag 기반 캐싱 또는 Redis 캐시
```java
private ExchangeFilterFunction logRequest() {
    return (request, next) -> {
        log.info("➡️ Request: {} {}", request.method(), request.url());
        request.headers().forEach((n, v) -> log.debug("{}: {}", n, v));
        return next.exchange(request);
    };
}

private ExchangeFilterFunction logResponse() {
    return ExchangeFilterFunction.ofResponseProcessor(response -> {
        log.info("⬅️ Response: {}", response.statusCode());
        return Mono.just(response);
    });
}
```
<br/>

### 2-4. 추가 설정

 - `SSL / 인증서 신뢰 무시`
```java
SslContext sslContext = SslContextBuilder.forClient()
        .trustManager(InsecureTrustManagerFactory.INSTANCE) // 신뢰하지 않아도 통과
        .build();

HttpClient secureHttpClient = HttpClient.create()
        .secure(spec -> spec.sslContext(sslContext));

WebClient webClient = WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(secureHttpClient))
        .build();
```
<br/>

 - `압축 / 자동 리다이렉트 설정`
```java
HttpClient httpClient = HttpClient.create()
    .compress(true) // Gzip 등 압축 자동 해제
    .followRedirect(true);
```
<br/>

 - `로깅 레벨별 제어`
```yml
logging:
  level:
    reactor.netty.http.client: DEBUG
    org.springframework.web.reactive.function.client: TRACE
```
<br/>

 - `커스텀 전송 전략 (backpressure, keep-alive, etc.)`
```java
HttpClient httpClient = HttpClient.create()
    .keepAlive(true)
    .followRedirect(true)
    .metrics(true)
    .wiretap("reactor.netty.http.client", LogLevel.DEBUG, AdvancedByteBufFormat.TEXTUAL);
```
<br/>

## 3. 필터(ExchangeFilterFunction)와 체이닝 (충돌/우선순위)

 - `헤더/쿠키/URI` 는 __나중에 적용한 쪽이 최종값__ 이 된다.
    - 필터는 ClientRequest를 전송 직전에 받아 복제/변경할 수 있기 때문
 - `필터 실행 순서`는 __등록한 순서대로 실행__ 된다.
    - 가장 마지막에 등록한 필터가 같은 헤더를 set하면 앞에서 설정한 값을 덮어쓴다.
 - `요청 체인에서 지정한 헤더`는 이미 __ClientRequest에 반영된 상태로 필터에게 전달__
    - 필터가 `ClientRequest.from(req).headers(h -> h.set("Authorization", "..."))`처럼 명시적으로 덮어쓰면 필터 값이 최종 적용됩니다.
 - `baseUrl과 절대/상대 URI`
    - `baseUrl("https://api.example.com")`를 써도, __요청에서 절대 URL을 쓰면 절대 URL이 우선__
    - 상대 경로("/v1/items")는 baseUrl과 합쳐진다.
 - `retrieve().onStatus(...) vs 필터의 예외 처리`
    - 필터에서 예외를 `Mono.error(...)`로 던져버리면 아래 단계(`retrieve().onStatus(...)`)는 도달하지 못함. __필터의 예외 변환이 우선__
    - 필터가 단순 로깅만 하고 넘기면, `retrieve().onStatus(...)`가 상태별 예외 변환을 수행
    - 권장 패턴: 공통 정책(인증 토큰 주입, 공통 로깅/메트릭, 공통 리트라이 전처리)은 필터에서 엔드포인트별 세부 오류 매핑은 `retrieve().onStatus(...)`에서 처리한다.

<br/>

### 3-1. 충돌 사례 예시

 - `헤더 덮어쓰기`
    - 필터 값이 헤더로 전달된다.
```java
// 1) 필터 등록
WebClient client = WebClient.builder()
    .filter((request, next) -> {
        ClientRequest mutated = ClientRequest.from(request)
            // 나중에 set 하면 최종값이 됨
            .headers(h -> h.set("Authorization", "Bearer FILTER_TOKEN"))
            .build();
        return next.exchange(mutated);
    })
    .build();

// 2) 요청에서 헤더 지정
client.get()
    .uri("https://example.com")
    .header("Authorization", "Bearer REQUEST_TOKEN")
    .retrieve()
    .toBodilessEntity()
    .subscribe();
```
<br/>

 - `응답 필터에서 예외를 던진 경우`
    - 응답 필터에서 예외를 던진 경우 retrieve().onStatus()에 도달하지 않는다.
```java
@Bean
WebClient shortCircuit(WebClient.Builder builder) {
    return builder
        .filter(ExchangeFilterFunction.ofResponseProcessor(res -> {
            if (res.statusCode().isError()) {
                // 여기서 스트림을 끝내버림 → 아래 retrieve/onStatus는 실행되지 않음
                return res.createException().flatMap(Mono::error);
            }
            return Mono.just(res);
        }))
        .build();
}

// 사용
webClient.get().uri("/api/secure")
    .retrieve()
    .onStatus(s -> true, r -> Mono.error(new RuntimeException("never reached")))
    .bodyToMono(String.class);
```
<br/>

 - `응답 필터에서 예외를 던지지 않는 경우`
    - 필터는 그냥 통과 → retrieve().onStatus(...)가 정상적으로 상태별 예외 변환 수행
    - 충돌 없음
```java
@Bean
WebClient okPassThrough(WebClient.Builder builder) {
    return builder
        .filter(ExchangeFilterFunction.ofResponseProcessor(res -> {
            // 상태/헤더만 로깅 — 바디는 손대지 않음(중요)
            log.info("⬅ status={} {}", res.statusCode(), res.headers().asHttpHeaders());
            return Mono.just(res);  // 그대로 아래 체인으로 전달
        }))
        .build();
}

// 사용
webClient.get().uri("/api")
    .retrieve()
    .onStatus(HttpStatusCode::is4xxClientError, r -> Mono.error(new My4xxException()))
    .onStatus(HttpStatusCode::is5xxServerError, r -> Mono.error(new My5xxException()))
    .bodyToMono(String.class);
```
<br/>

 - `응답 필터에서 Body를 읽는 경우`
    - 필터에서 바디를 사용하면, 해당 스트림이 비어있게 된다. 때문에, 다운스트림은 바디를 더 이상 읽지 못하게 된다.
    - 필터에서 바디를 읽으면 반드시 복구해서 내려보내야 함. 단, 이 방식은 메모리에 바디 전체를 담는 방식이라 SSE/NDJSON/대용량에는 절대 사용하지 말 것.
```java
// ❌ 잘못된 예 (바디를 읽고 그냥 버림 — 다운스트림은 바디를 더 못 읽음)
.filter(ExchangeFilterFunction.ofResponseProcessor(res -> {
    return res.bodyToMono(String.class)
        .doOnNext(body -> log.debug("body = {}", body))
        .thenReturn(res); // 이미 소비됨. 아래에서 bodyToMono/Flux 하면 Empty/에러
}))

// ✅ 바디를 복제해 다시 채워 넣는 올바른 패턴(단, 스트리밍에는 금지)
.filter(ExchangeFilterFunction.ofResponseProcessor(res -> {
    // ※ 대용량/스트리밍에는 위험. 일반 JSON 단건 응답에서만 사용 권장.
    return res.bodyToMono(String.class)
        .defaultIfEmpty("")
        .flatMap(body -> {
            log.debug("body = {}", body);
            if (!body.isEmpty()) {
                // Body가 너무 길면 일부만 로깅
                if (body.length() > 1000) {
                    log.info("Response Body: {}... (truncated, total length: {})",
                            body.substring(0, 1000), body.length());
                } else {
                    log.info("Response Body: {}", body);
                }
            } else {
                log.info("Response Body: [Empty]");
            }

            // 바디를 다시 채워넣어 downstream이 또 읽을 수 있게 만들기
            ClientResponse mutated = res.mutate()
                .body(BodyInserters.fromValue(body)) // 바디 복구
                .build();

            return Mono.just(mutated);
        });
}))
```
<br/>

## 4. 스트리밍 처리

### 4-1. 주요 설정 체크리스트

 - `커넥션/타임아웃(React Netty)`
    - __연결(Connect) 타임아웃__: 너무 길게 두지 않기 (예: 3s)
    - __응답(Response) 타임아웃__: 무한 스트림(SSE/NDJSON)은 설정하지 않음
        - → 대신 ReadTimeoutHandler로 idle(무응답)만 감시
    - __WriteTimeoutHandler__: 업스트림으로 쓰기 지연 보호
    - __커넥션 풀__: 스트리밍 전용 풀(이름/최대 커넥션 별도) 권장
 - `코덱/버퍼`
    - `maxInMemorySize`는 __소폭 상향(예: 512KB)__ 정도면 충분(대용량 단건 받지 않음)
    - NDJSON은 줄 단위 파싱, SSE는 text/event-stream로 라인 단위 소비
 - `로깅`
    - 바디 로깅 금지(스트림 소진)
    - → 상태/헤더만 로그, 필요 시 wiretap은 개발에서만
 - `서버 반환 헤더`
    - SSE: produces = text/event-stream
    - NDJSON: produces = application/x-ndjson
    - 압축: compress(true) 사용 가능
 - `안정성 옵션`
    - 하트비트(서버→클라): :\n 또는 ping 이벤트 정기 전송
    - 재연결: 클라이언트에서 exponential backoff
    - 역압(Backpressure): 그대로 Flux 반환(서블릿 스택에서도 비점유)

<br/>

### 4-2. WebClient 설정 및 사용 예시

```java
@Slf4j
@Configuration
public class StreamingWebClientConfig {

    @Bean
    public WebClient streamingWebClient(ObjectMapper om) {
        ConnectionProvider pool = ConnectionProvider.builder("stream-pool")
            .maxConnections(200)
            .maxIdleTime(Duration.ofSeconds(30))
            .maxLifeTime(Duration.ofMinutes(5))
            .pendingAcquireTimeout(Duration.ofSeconds(10))
            .evictInBackground(Duration.ofSeconds(60))
            .build();

        HttpClient http = HttpClient.create(pool)
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
            // responseTimeout은 무한스트림에서는 사용하지 않음
            .doOnConnected(conn -> conn
                .addHandlerLast(new ReadTimeoutHandler(60))   // idle read 60s
                .addHandlerLast(new WriteTimeoutHandler(30))) // write 30s
            .keepAlive(true)
            .compress(true);

        ExchangeStrategies strategies = ExchangeStrategies.builder()
            .codecs(c -> {
                c.defaultCodecs().maxInMemorySize(512 * 1024); // 512KB
                c.defaultCodecs().jackson2JsonEncoder(new Jackson2JsonEncoder(om, MediaType.APPLICATION_JSON));
                c.defaultCodecs().jackson2JsonDecoder(new Jackson2JsonDecoder(om, MediaType.APPLICATION_JSON));
            })
            .build();

        return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(http))
            .exchangeStrategies(strategies)
            .filter(ExchangeFilterFunction.ofResponseProcessor(res -> {
                // 바디는 건드리지 말고 상태/헤더만
                log.info("[Stream] status={} headers={}", res.statusCode(), res.headers().asHttpHeaders());
                return Mono.just(res);
            }))
            .build();
    }
}
```
<br/>

 - `업스트림 소비 유틸 (SSE / NDJSON)`
    - 필요하면 안정성 강화: `.retryWhen(Retry.backoff(5, Duration.ofSeconds(1)).maxBackoff(Duration.ofSeconds(30)).jitter(0.5)).onErrorResume(e -> Flux.empty())` 등
```java
@Component
@RequiredArgsConstructor
public class UpstreamStreamClient {
    private final WebClient streamingWebClient;

    /** SSE: 원시 라인(문자열)으로 수신 */
    public Flux<String> sse(URI uri) {
        return streamingWebClient.get()
            .uri(uri)
            .accept(MediaType.TEXT_EVENT_STREAM)
            .exchangeToFlux(res -> {
                if (res.statusCode().is2xxSuccessful())
                    return res.bodyToFlux(String.class);
                return res.createException().flatMapMany(Flux::error);
            })
            .name("sse-stream")
            .doOnSubscribe(s -> log.info("[SSE] subscribe {}", uri))
            .doOnError(e -> log.warn("[SSE] error {}", e.toString()))
            .doFinally(sig -> log.info("[SSE] finally {}", sig));
    }

    /** NDJSON: 줄단위 JSON → DTO (모르면 JsonNode로 안전 수신) */
    public <T> Flux<T> ndjson(URI uri, Class<T> type) {
        return streamingWebClient.get()
            .uri(uri)
            .accept(MediaType.APPLICATION_NDJSON)
            .exchangeToFlux(res -> {
                if (res.statusCode().is2xxSuccessful())
                    return res.bodyToFlux(type);
                return res.createException().flatMapMany(Flux::error);
            })
            .name("ndjson-stream")
            .doOnSubscribe(s -> log.info("[NDJSON] subscribe {}", uri))
            .doOnError(e -> log.warn("[NDJSON] error {}", e.toString()))
            .doFinally(sig -> log.info("[NDJSON] finally {}", sig));
    }
}
```
<br/>

 - `프록시 컨트롤러(서버도 스트리밍으로 내려주기)`
```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/proxy")
public class StreamProxyController {
    private final UpstreamStreamClient upstream;

    /** SSE → SSE 프록시 */
    @GetMapping(value = "/sse", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> proxySse(@RequestParam String target) {
        return upstream.sse(URI.create(target))
            // 필요 시 하트비트 보완: 25초간 라인이 없으면 keepalive
            .timeout(Duration.ofSeconds(30))
            .onErrorResume(TimeoutException.class, e ->
                Flux.just(":\n")) // SSE 주석 하트비트 한 번 내보내고 이어감
            .repeat(); // 단순 예시: 재시도 전략은 환경에 맞게
    }

    /** NDJSON → NDJSON 프록시 (그대로 문자열 라인으로) */
    @GetMapping(value = "/ndjson", produces = "application/x-ndjson")
    public Flux<String> proxyNdjson(@RequestParam String target) {
        return upstream.ndjson(URI.create(target), com.fasterxml.jackson.databind.JsonNode.class)
            .map(node -> node.toString()); // 한 줄 한 객체
    }

    /** (선택) SSE 포맷 엄격히: ServerSentEvent<T> */
    @GetMapping(value = "/sse/strict", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<org.springframework.http.codec.ServerSentEvent<String>> proxySseStrict(@RequestParam String target) {
        return upstream.sse(URI.create(target))
            .map(line -> org.springframework.http.codec.ServerSentEvent.<String>builder()
                .event("message")
                .data(line)
                .build());
    }
}
```
<br/>

 - `프론트 엔드`
```javascript
// SSE
const url = '/proxy/sse?target=' + encodeURIComponent('https://upstream/sse');
const es = new EventSource(url);
es.onmessage = (e) => console.log('SSE:', e.data);
es.onerror = (e) => console.warn('SSE error', e);

// NDJSON
(async () => {
  const res = await fetch('/proxy/ndjson?target=' + encodeURIComponent('https://upstream/ndjson'));
  const reader = res.body.getReader();
  const decoder = new TextDecoder();
  let buf = '';
  for (;;) {
    const { value, done } = await reader.read();
    if (done) break;
    buf += decoder.decode(value, { stream: true });
    let idx;
    while ((idx = buf.indexOf('\n')) >= 0) {
      const line = buf.slice(0, idx).trim();
      buf = buf.slice(idx + 1);
      if (line) console.log('NDJSON item:', JSON.parse(line));
    }
  }
})();
```
