# RestTemplate

RestTemplate은 Spring Framework에서 제공하는 HTTP 통신을 간편하게 처리하기 위한 클래스입니다. 이를 통해 서버 간의 통신, 특히 RESTful 웹 서비스와 상호작용하는 데 매우 유용하며, HTTP 요청을 전송하고 응답을 처리할 수 있습니다. RestTemplate은 여러 HTTP 메서드(GET, POST, PUT, DELETE 등)를 지원하며, 다양한 방식으로 요청 및 응답을 처리할 수 있도록 다양한 메서드를 제공합니다.  

 - HTTP 요청 메서드 지원: GET, POST, PUT, DELETE, PATCH 등의 메서드를 사용하여 REST API와 통신할 수 있습니다.
 - 동기 방식: RestTemplate은 동기적으로 동작하므로, 요청이 완료될 때까지 블로킹됩니다.
 - 다양한 응답 형식 처리: JSON, XML 등 다양한 응답 형식을 처리할 수 있으며, 이를 Java 객체로 변환할 수 있는 기능도 제공합니다.
 - 간단한 오류 처리: 기본적으로 HTTP 응답 상태 코드에 따라 예외를 던지며, 이를 통해 간단한 오류 처리가 가능합니다.
 - URI 템플릿 지원: URI에 변수를 전달할 수 있는 기능을 제공하여, 가변적인 URL을 처리할 수 있습니다.

## RestTemplate 주요 메서드

 - GET 요청
```java
// 주어진 URL로 GET 요청을 보내고, 응답을 특정 객체로 변환합니다.
<T> T getForObject(URI url, Class<T> responseType);
<T> T getForObject(String url, Class<T> responseType, Object... uriVariables);
<T> T getForObject(String url, Class<T> responseType, Map<String, ?> uriVariables);

// 응답을 ResponseEntity<T> 타입으로 반환하여, HTTP 상태 코드 및 헤더 정보까지 함께 처리할 수 있습니다.
<T> ResponseEntity<T> getForEntity(String url, Class<T> responseType, Object... uriVariables);
<T> ResponseEntity<T> getForEntity(String url, Class<T> responseType, Map<String, ?> uriVariables);
<T> ResponseEntity<T> getForEntity(URI url, Class<T> responseType);

// GET 요청 - URL 파라미터 전달
String url = "https://api.example.com/data/{id}";
Map<String, String> params = new HashMap<>();
params.put("id", "123");

ResponseEntity<String> response = restTemplate.getForEntity(url, String.class, params);

if (response.getStatusCode() == HttpStatus.OK) {
    System.out.println("Response: " + response.getBody());
}
```

 - POST 요청
```java
// POST 요청을 보내고, 응답을 특정 객체로 변환합니다.
<T> T postForObject(String url, @Nullable Object request, Class<T> responseType, Object... uriVariables);
<T> T postForObject(String url, @Nullable Object request, Class<T> responseType, Map<String, ?> uriVariables);
<T> T postForObject(URI url, @Nullable Object request, Class<T> responseType);

// ResponseEntity<T> 형태로 응답을 받습니다.
<T> ResponseEntity<T> postForEntity(String url, @Nullable Object request, Class<T> responseType, Object... uriVariables);
<T> ResponseEntity<T> postForEntity(String url, @Nullable Object request, Class<T> responseType, Map<String, ?> uriVariables);
<T> ResponseEntity<T> postForEntity(URI url, @Nullable Object request, Class<T> responseType);

// POST 요청 - 요청 본문에 데이터 전달
String url = "https://api.example.com/data";
Map<String, Object> request = new HashMap<>();
request.put("name", "John Doe");
request.put("age", 30);

ResponseEntity<String> response = restTemplate.postForEntity(url, request, String.class);

if (response.getStatusCode() == HttpStatus.CREATED) {
    System.out.println("Created: " + response.getBody());
}
```

 - PUT & PATCH & DELETE 요청
```java
// 주어진 URL로 PUT 요청을 보냅니다.
void put(String url, @Nullable Object request, Object... uriVariables);
void put(String url, @Nullable Object request, Map<String, ?> uriVariables);
void put(URI url, @Nullable Object request);

// 주어진 URL로 PATCH 요청을 보내고, 응답을 특정 객체로 변환합니다.
<T> T patchForObject(String url, @Nullable Object request, Class<T> responseType, Object... uriVariables);
<T> T patchForObject(String url, @Nullable Object request, Class<T> responseType, Map<String, ?> uriVariables);
<T> T patchForObject(URI url, @Nullable Object request, Class<T> responseType);

// 주어진 URL로 DELETE 요청을 보냅니다.
void delete(String url, Object... uriVariables)
void delete(String url, Map<String, ?> uriVariables)
void delete(URI url)
```

 - HTTP 요청
    - exchange() 메서드를 사용하면 모든 HTTP 메서드에 대한 요청을 처리할 수 있다.
    - exchange()는 요청 메서드, 헤더, 요청 본문 등을 세밀하게 조정할 수 있다.
    - __HTTP 메서드 지정 가능__: GET, POST, PUT, DELETE 등 모든 HTTP 메서드를 사용할 수 있습니다.
    - __헤더 설정 가능__: 요청에 대한 헤더 설정을 지원합니다. 이를 통해 인증, 캐시 제어 등을 할 수 있습니다.
    - __요청 본문 전송__: POST나 PUT 같은 메서드에서 요청 본문에 데이터를 보낼 수 있습니다.
    - __응답 처리__: 응답은 ResponseEntity<T> 객체로 반환되며, 상태 코드, 헤더, 본문을 모두 처리할 수 있습니다.
    - __요청 주요 클래스__
        - HttpEntity 클래스에는 Header와 Body가 포함된다.
        - RequestEntity 클래스에는 HTTP Method, URI, Header, Body가 포함된다.
    - __요청 방법__
        - URL, HTTP Method, HttpEntity(Header, Body), 응답 Class를 하나씩 설정해서 보내기
        - RequestEntity(URL, HTTP Method, HttpEntity)와 응답 Class를 설정해서 보내기
```java
// exchange() 메서드 시그니처
<T> ResponseEntity<T> exchange(String url, HttpMethod method, @Nullable HttpEntity<?> requestEntity, Class<T> responseType, Object... uriVariables);
<T> ResponseEntity<T> exchange(String url, HttpMethod method, @Nullable HttpEntity<?> requestEntity, Class<T> responseType, Map<String, ?> uriVariables);
<T> ResponseEntity<T> exchange(URI url, HttpMethod method, @Nullable HttpEntity<?> requestEntity, Class<T> responseType);
<T> ResponseEntity<T> exchange(String url, HttpMethod method, @Nullable HttpEntity<?> requestEntity, ParameterizedTypeReference<T> responseType, Object... uriVariables);
<T> ResponseEntity<T> exchange(String url, HttpMethod method, @Nullable HttpEntity<?> requestEntity, ParameterizedTypeReference<T> responseType, Map<String, ?> uriVariables);
<T> ResponseEntity<T> exchange(URI url, HttpMethod method, @Nullable HttpEntity<?> requestEntity, ParameterizedTypeReference<T> responseType);
<T> ResponseEntity<T> exchange(RequestEntity<?> requestEntity, Class<T> responseType);
<T> ResponseEntity<T> exchange(RequestEntity<?> requestEntity, ParameterizedTypeReference<T> responseType);
```

### exchange() 메서드 예제

 - URI 객체 만들기
    - UriComponentsBuilder
        - UriComponentsBuilder는 복잡한 URI를 쉽게 만들 수 있도록 도와주는 Spring 제공 유틸리티 클래스입니다. 특히 URL 매개변수를 포함한 동적인 URI 생성에 유용합니다.
    - URI.create()
        - URI.create()는 간단한 URI 문자열을 직접 URI 객체로 변환할 때 사용합니다. URI가 고정되어 있거나 간단한 경우에 적합합니다.
    - new URI()
        - URI 클래스의 생성자를 사용해서 개별적으로 스킴, 호스트, 경로 등을 지정할 수 있습니다.
```java
// UriComponentsBuilder
URI uri = UriComponentsBuilder.newInstance()
    .fromHttpUrl("https:www.example.com/api/v1/resource/{id}")
    .queryParam("sort", "desc")
    .buildAndExpand("1")
    .toUri();

URI uri = UriComponentsBuilder.newInstance()
    .scheme("https")
    .host("www.example.com")
    .path("/api/v1/resource/{id}")
    .queryParam("sort", "desc")
    .buildAndExpand("1")
    .toUri();

// URI.create()
URI uri = URI.create("https://www.example.com/api/v1/resource");

// new URI()
URI uri = new URI("https", "www.example.com", "/api/v1/resource", "sort=desc", null);
```

 - HttpEntity & RequestEntity 만들기
    - HttpEntity에는 Header와 Body를 설정할 수 있다.
        - 팩토리 메서드가 지원되지 않는다. new 키워드로 생성자에서 설정할 수 있다.
        - 헤더를 설정하기 위해서는 MultiValueMap을 인자로 넘겨야 하는데, MultiValueMap 에도 팩토리 메서드가 지원되지 않는다.
    - RequestEntity에는 URI, Method, Header, Body를 설정할 수 있다.
```java
// HttpEntity - 헤더와 본문
MultiValueMap<String, String> headers = new HttpHeaders();
headers.add("Authorization", "Bearer my-token");
headers.add("Custom-Header", "CustomValue");

HttpEntity<XxxClientRequest> httpEntity = new HttpEntity<>(xxxClientRequest, headers);

// RequestEntity - POST 요청(본문과 헤더)
RequestEntity<XxxClientRequeset> requestEntity = RequestEntity.post(uri)
    .headers(httpHeaders -> {
        httpHeaders.add("Content-Type", "application/json");
    })
    .body(xxxClientrequest);
```

 - exchange 요청
    - HttpEntity 사용시
        - URI와 MTTP Method를 매개변수로 넘겨주어야 한다.
        - 만약, Body와 Header가 필요없는 경우 HttpEntity 매개변수에 null을 넘긴다.
    - RequestEntity 사용시
        - RequestEntity안에 이미 URI와 HTTP Method가 정의되어 있다.
        - 때문에, 반환 타입만 매개변수에 넘겨주면 된다.
```java
// HttpEntity 사용시
ResponseEntity<String> response = restTemplate.exchange(uri, HttpMethod.POST, httpEntity, String.class);

// RequestEntity 사용시
ResponseEntity<String> response = restTemplate.exchange(requestEntity, String.class);
```

## RestTemplate 내부 코드

 - org.springframework.web.client.RestTemplate
    - 생성자 안에서 기본 예외 핸들러를 설정한다.
    - xxxForEntity(), exchange() 메서드 모두 execute() 메서드를 호출한다.
    - execute() 내부에서는 doExecute() 메서드를 호출하며, doExecute() 메서드에서는 요청을 수행한 후 handleResponse() 메서드를 수행한다.
        - handleResponse() 내부에서 errorHandler 객체로 에러 검사를 진행하는데, 기본적으도 DefaultResponseErrorHandler가 사용된다.
        - DefaultResponseErrorHandler는 기본적으로 4xx, 5xx 응답에 대해서 hasError()를 true로 반환하고, handleError() 에서 예외를 발생시킨다.
```java
// 생성자
public RestTemplate() {
    // 기본 메시지 컨버터 설정
    this.messageConverters = new ArrayList();
    this.messageConverters.add(new ByteArrayHttpMessageConverter());
    this.messageConverters.add(new StringHttpMessageConverter());
    // ..

    // 기본 예외 핸들러 설정
    this.errorHandler = new DefaultResponseErrorHandler();

    // ..
}

public <T> T xxxForEntity(..) {
    RequestCallback requestCallback = this.acceptHeaderRequestCallback(responseType);
    ResponseExtractor<ResponseEntity<T>> responseExtractor = this.responseEntityExtractor(responseType);
    return this.execute(..);
}

public <T> ResponseEntity<T> exchange(..) {
    RequestCallback requestCallback = this.httpEntityCallback(requestEntity, responseType);
    ResponseExtractor<ResponseEntity<T>> responseExtractor = this.responseEntityExtractor(responseType);
    return this.execute(..);
}

// doExecute
protected <T> T doExecute(..) {

    // 요청 객체 초기화
    ClientHttpRequest request;
    try {
        request = this.createRequest(url, method);
    } catch (IOException var21) {
        IOException ex = var21;
        throw createResourceAccessException(url, method, ex);
    }

    // 요청 수행
    try {
        Observation.Scope scope = observation.openScope();

        try {
            if (requestCallback != null) {
                requestCallback.doWithRequest(request);
            }

            response = request.execute();
            observationContext.setResponse(response);

            // 요청 이후에 handleResponse() 수행
            this.handleResponse(url, method, response);
            var28 = responseExtractor != null ? responseExtractor.extractData(response) : null;
        } catch (Throwable var22) {
            // ..
        }
    } catch (IOException var23) {
        // ..
    } finally {
        // ..
    }
}

// 응답 핸들링
protected void handleResponse(..) {
    ResponseErrorHandler errorHandler = this.getErrorHandler();
    boolean hasError = errorHandler.hasError(response);
    // ..

    if (hasError) {
        // errorHandler의 hasError() 메서드를 수행하고, 예외가 존재하는 경우 예외 핸들링
        errorHandler.handleError(url, method, response);
    }

}

/**
 * DefaultResponseErrorHandler
 */
public class DefaultResponseErrorHandler implements ResponseErrorHandler {

    public boolean hasError(ClientHttpResponse response) throws IOException {
        HttpStatusCode statusCode = response.getStatusCode();
        return this.hasError(statusCode);
    }

    // 4xx, 5xx 상태 코드는 true, 나머지(1xx, 2xx, 3xx)는 false
    protected boolean hasError(HttpStatusCode statusCode) {
        return statusCode.isError();
    }

    // 4xx는 HttpClientErrorException
    // 5xx는 HttpServerErrorException
    // 그 외 UnknownHttpStatusCodeException (hasError 이후에 수행되어 발생할 일이 없기는 하다.)
    protected void handleError(..) throws IOException {
        String statusText = response.getStatusText();
        HttpHeaders headers = response.getHeaders();
        byte[] body = this.getResponseBody(response);
        Charset charset = this.getCharset(response);
        String message = this.getErrorMessage(statusCode.value(), statusText, body, charset);
        Object ex;
        if (statusCode.is4xxClientError()) {
            ex = HttpClientErrorException.create(message, statusCode, statusText, headers, body, charset);
        } else if (statusCode.is5xxServerError()) {
            ex = HttpServerErrorException.create(message, statusCode, statusText, headers, body, charset);
        } else {
            ex = new UnknownHttpStatusCodeException(message, statusCode.value(), statusText, headers, body, charset);
        }

        if (!CollectionUtils.isEmpty(this.messageConverters)) {
            ((RestClientResponseException)ex).setBodyConvertFunction(this.initBodyConvertFunction(response, body));
        }

        throw ex;
    }
}
```

## RestTemplate 예외 핸들링

RestTemplate을 사용하여 HTTP Client 요청을 보내고, 응답이 4xx, 5xx로 오면 자동으로 예외가 발생하기 떄문에, 예외 핸들링이 필요하다. 요청을 수행하는 부분에 try, catch로 예외 핸들링을 할 수도 있고, ResponseErrorHandler 구현체를 직접 만들어서 응답에 대해서 어떻게 처리할 지 직접 정의해 줄 수도 있다.  
예외 발생시 재시도 수행 같은 기능은 제공되지 않는데, Spring Retry와 통합하면 좀 더 쉽게 구현할 수 있다.  

### 첫 번째 개선. 예외 핸들링

```java
try {
    ResponseEntity<Map> response = restTemplate.exchange(requestEntity, Map.class);
    // 정상 응답 처리
} catch (HttpClientErrorException e) {
    // 4xx 예외 응답 처리
} catch (HttpServerErrorException e) {
    // 5xx 예외 응답 처리
} catch (RestClientException e) {
    // 그 외 예외 응답 처리
}
```

### 두 번째 개선. 공통 예외 처리 핸들러 & 커스텀 예외 도입

일관된 응답 포맷(ApiResponse)을 사용하여 성공과 실패 시 동일한 구조로 응답을 반환하게 한다.  
중앙 집중식 예외 처리(@ControllerAdvice)를 사용하여 모든 예외를 글로벌하게 처리하므로 예외 처리가 더 체계적이고 관리가 용이하게 한다.  
커스텀 예외 도입(ApiClientException, ApiServerException, ApiUnknownException) 하여 각 예외를 구체적으로 처리하며, 필요 시 확장할 수 있도록 한다.  


 - 공통 응답 클래스
    - 응답을 관리하는 공통 포맷을 사용하여 성공 및 실패 응답을 일관되게 처리할 수 있습니다.
```java
public class ApiResponse {

    private boolean success;
    private Object data;
    private String errorMessage;

    // 성공적인 응답을 위한 팩토리 메서드
    public static ApiResponse success(Object data) {
        ApiResponse response = new ApiResponse();
        response.success = true;
        response.data = data;
        return response;
    }

    // 실패 응답을 위한 팩토리 메서드
    public static ApiResponse error(String errorMessage) {
        ApiResponse response = new ApiResponse();
        response.success = false;
        response.errorMessage = errorMessage;
        return response;
    }

    // getters and setters...
}
```

 - 커스텀 예외 클래스
    - 클라이언트와 서버 오류를 명확하게 처리하기 위해 커스텀 예외를 정의합니다.
```java
public class ApiClientException extends RuntimeException {
    private HttpStatus statusCode;
    private String responseBody;

    public ApiClientException(HttpStatus statusCode, String responseBody) {
        super("Client error: " + statusCode);
        this.statusCode = statusCode;
        this.responseBody = responseBody;
    }

    public HttpStatus getStatusCode() {
        return statusCode;
    }

    public String getResponseBody() {
        return responseBody;
    }
}

public class ApiServerException extends RuntimeException {
    private HttpStatus statusCode;
    private String responseBody;

    public ApiServerException(HttpStatus statusCode, String responseBody) {
        super("Server error: " + statusCode);
        this.statusCode = statusCode;
        this.responseBody = responseBody;
    }

    public HttpStatus getStatusCode() {
        return statusCode;
    }

    public String getResponseBody() {
        return responseBody;
    }
}

public class ApiUnknownException extends RuntimeException {
    public ApiUnknownException(String message) {
        super("Unknown error occurred: " + message);
    }
}
```

 - RestTemplate 사용
    - 예외에 대해서 커스텀한 예외를 발생시키도록 한다.
```java
try {
    ResponseEntity<Map> response = restTemplate.exchange(requestEntity, Map.class);
    return ApiResponse.success(response.getBody());
} catch (HttpClientErrorException e) {
    throw new ApiClientException(e.getStatusCode(), e.getResponseBodyAsString());
} catch (HttpServerErrorException e) {
    throw new ApiServerException(e.getStatusCode(), e.getResponseBodyAsString());
} catch (RestClientException e) {
    throw new ApiUnknownException(e.getMessage());
}
```

 - 글로벌 예외 처리 핸들러
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ApiClientException.class)
    public ResponseEntity<ApiResponse> handleClientException(ApiClientException ex) {
        System.err.println("Client error: " + ex.getStatusCode() + " - " + ex.getResponseBody());
        return ResponseEntity
            .status(ex.getStatusCode())
            .body(ApiResponse.error("Client error occurred: " + ex.getStatusCode()));
    }

    @ExceptionHandler(ApiServerException.class)
    public ResponseEntity<ApiResponse> handleServerException(ApiServerException ex) {
        System.err.println("Server error: " + ex.getStatusCode() + " - " + ex.getResponseBody());
        return ResponseEntity
            .status(ex.getStatusCode())
            .body(ApiResponse.error("Server error occurred: " + ex.getStatusCode()));
    }

    @ExceptionHandler(ApiUnknownException.class)
    public ResponseEntity<ApiResponse> handleUnknownException(ApiUnknownException ex) {
        System.err.println("Unknown error: " + ex.getMessage());
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ApiResponse.error("An unknown error occurred"));
    }
}
```

### 세 번째 개선. ResponseErrorHandler 구현

RestTemplate을 사용하는 곳에 직접 try, catch를 이용하여 예외를 핸들링하고 커스텀 예외를 발생시키고 있다. ResponseErrorHandler 구현체를 등록하여, 응답에 따라 커스텀 예외를 발생시키도록 오버라이딩할 수 있다.  

 - ResponseErrorHandler 구현
```java
public class CustomResponseErrorHandler implements ResponseErrorHandler {

    @Override
    public boolean hasError(ClientHttpResponse response) throws IOException {
        // 응답이 에러인지 여부를 판단하는 로직 (4xx 또는 5xx 상태 코드일 때 true 반환)
        HttpStatus statusCode = response.getStatusCode();
        return (statusCode.is4xxClientError() || statusCode.is5xxServerError());
    }

    @Override
    public void handleError(ClientHttpResponse response) throws IOException {
        HttpStatus statusCode = response.getStatusCode();
        String responseBody = StreamUtils.copyToString(response.getBody(), StandardCharsets.UTF_8);

        if (statusCode.is4xxClientError()) {
            handleClientError(statusCode, responseBody);
        } else if (statusCode.is5xxServerError()) {
            handleServerError(statusCode, responseBody);
        } else {
            throw new ApiUnknownException("Unknown error occurred: " + statusCode);
        }
    }

    // 클라이언트 오류(4xx) 처리
    private void handleClientError(HttpStatus statusCode, String responseBody) throws ApiClientException {
        System.err.println("Client error: " + statusCode + " - " + responseBody);
        throw new ApiClientException(statusCode, responseBody);
    }

    // 서버 오류(5xx) 처리
    private void handleServerError(HttpStatus statusCode, String responseBody) throws ApiServerException {
        System.err.println("Server error: " + statusCode + " - " + responseBody);
        throw new ApiServerException(statusCode, responseBody);
    }
}
```

 - RestTemplate에 예외 핸들러 등록
```java
@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate() {
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.setErrorHandler(new CustomResponseErrorHandler());
        return restTemplate;
    }
}
```

## RestTemplate 활용

 - 벤더사마다 다르게 빈 등록
    - connection timeout 설정이 각기 다릅니다.
    - 로깅을 각기 다르게 설정 할 수 있습니다.
    - 예외 처리가 각기 다릅니다.
    - API에 대한 권한 인증이 각기 다릅니다.
```java
@Bean
public RestTemplate localTestTemplate() {
return restTemplateBuilder.rootUri("http://localhost:8899")
    .additionalInterceptors(new RestTemplateClientHttpRequestInterceptor())
    .errorHandler(new RestTemplateErrorHandler())
    .setConnectTimeout(Duration.ofMinutes(3))
    .build();
}


@Bean
public RestTemplate xxxPaymentTemplate() {
return restTemplateBuilder.rootUri("http://xxxx")
    .additionalInterceptors(new RestTemplateClientHttpRequestInterceptor())
    .errorHandler(new RestTemplateErrorHandler())
    .setConnectTimeout(Duration.ofMinutes(3))
    .build();
}
```

 - 로깅
    - restTemplateBuilder의 additionalInterceptors() 메서드를 이용하면 로킹을 쉽게 구현할 수 있고 특정 Vendor의 Bean에는 더 구체적인 로킹, 그 이외의 작업을 Interceptors을 편리하게 등록할 수 있습니다.
    - Request, Response의 Logging을 저장하는 Interceptor 코드입니다. 결제와 같은 중요한 API 호출은 모든 요청과 응답을 모두 로킹 하는 것이 바람직합니다.
    - 상대적으로 덜 중요한 API 호출 같은 경우에는 Interceptor 등록하지 않아도 됩니다. 이처럼 Vendor 사마다 Bean으로 지정해서 관리하는 것이 효율적입니다.
```java
@Slf4j
public class RestTemplateClientHttpRequestInterceptor implements ClientHttpRequestInterceptor {

  @NonNull
  @Override
  public ClientHttpResponse intercept(@NonNull final HttpRequest request,
      @NonNull final byte[] body, final @NonNull ClientHttpRequestExecution execution)
      throws IOException {

    loggingRequest(request, body);

    final ClientHttpResponse response = execution.execute(request, body);

    loggingResponse(response);
    return response
  }

  private void loggingRequest(final HttpRequest request, byte[] body) {
    log.info("=====Request======");
    log.info("Headers: {}", request.getHeaders());
    log.info("Request Method: {}", request.getMethod());
    log.info("Request URI: {}", request.getURI());
    log.info("Request body: {}",
        body.length == 0 ? null : new String(body, StandardCharsets.UTF_8));
    log.info("=====Request======");
  }

  private void loggingResponse(ClientHttpResponse response) throws IOException {

    final String body = getBody(response);

    log.info("======Response=======");
    log.info("Headers: {}", response.getHeaders());
    log.info("Response Status : {}", response.getRawStatusCode());
    log.info("Request body: {}", body);
    log.info("======Response=======");

  }

  private String getBody(@NonNull final ClientHttpResponse response) throws IOException {
    try (BufferedReader br = new BufferedReader(new InputStreamReader(response.getBody()))) {
      return br.readLine();
    }
  }
}
```

 - 예외 처리
```java
public class RestTemplateErrorHandler implements ResponseErrorHandler {

  @Override
  public boolean hasError(@NonNull final ClientHttpResponse response) throws IOException {
    // response.getBody() 넘겨 받은 body 값으로 적절한 예외 상태 확인 이후 boolean return
    final HttpStatus statusCode = response.getStatusCode();
    return !statusCode.is2xxSuccessful();
  }

  // hasError에서 true를 return하면 해당 메서드 실행.
  @Override
  public void handleError(@NonNull final ClientHttpResponse response) throws IOException {
    // 상황에 알맞는 Error handling 로직 작성....
  }
```

