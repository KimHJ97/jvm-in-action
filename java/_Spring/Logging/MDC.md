# MDC (Mapped Diagnostic Context)

MDC는 Java에서 로그에 컨텍스트(Context)를 추가하기 위해 제공되는 기능입니다. 주로 SLF4J와 Logback 또는 Log4j 같은 로깅 프레임워크와 함께 사용됩니다. 이를 활용하면, 같은 스레드에서 실행되는 로그 메시지에 특정 정보를 추가하여 구체적인 컨텍스트를 유지할 수 있습니다.  

MDC(Mapped Diagnostic Context)는 현재 실행중인 쓰레드에 메타 정보를 넣고 관리하는 공간이다. MDC는 내부적으로 Map을 관리하고 있어 (Key, Value) 형태로 값을 저장할 수 있다. 메티 정보를 쓰레드 별로 관리하기 위해 내부적으로는 쓰레드 로컬을 사용하고 있다.  

 - __Thread-local storage 기반__: MDC는 각 스레드에서 독립적으로 값을 저장합니다. 즉, 한 스레드에서 설정한 값은 다른 스레드에 영향을 미치지 않습니다.
 - __추가적인 로그 컨텍스트 제공__: 예를 들어, 요청 ID, 사용자 ID, 트랜잭션 ID 같은 정보를 로그에 자동으로 추가할 수 있습니다.
 - __편리한 로그 필터링__: 로그를 분석하거나 필터링할 때 특정 컨텍스트 정보를 기준으로 로그를 추적할 수 있습니다.
 - __HTTP 요청 처리__: 각 HTTP 요청마다 고유의 요청 ID(Request ID)를 생성하고, 이를 로그에 포함하여 요청 단위로 로그를 추적할 수 있습니다.
 - __멀티스레드 환경__: 여러 스레드에서 동작하는 애플리케이션에서도 독립적으로 컨텍스트를 관리하여 로그를 구분할 수 있습니다.

## MDC 기본 사용법

 - `MDC 값 설정`
```java
import org.slf4j.MDC;

public class MdcExample {
    public static void main(String[] args) {
        // MDC에 값 설정
        MDC.put("requestId", "12345");
        MDC.put("userId", "john_doe");

        // 로그 출력
        System.out.println("This is a log message with context.");

        // MDC 값 제거
        MDC.clear();
    }
}
```

 - `로그백(Logback) 설정`
    - MDC 값을 로그에 포함하려면 로깅 설정 파일에서 `%X{key}` 를 사용합니다.
```xml
<!-- logback.xml -->
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%X{requestId}] [%X{userId}] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="debug">
        <appender-ref ref="CONSOLE" />
    </root>
</configuration>
```

## Spring Boot Filter에 MDC 적용

 - __doFilter 이후에 MDC를 clear 해주어야 한다.__ Spring MVC는 쓰레드 풀에 쓰레드들을 만들어놓고, 요청이 오면 쓰레드를 사용해 요청을 처리하고 반납한다. 때문에 clear를 해주지 않으면 다른 요청이 이 쓰레드를 재사용할 대 이전 데이터가 남아있을 수 있다.
```java
import org.slf4j.MDC;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.UUID;

@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class MdcFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        try {
            // 요청 ID 생성 및 MDC에 저장
            String requestId = UUID.randomUUID().toString();
            MDC.put("requestId", requestId);

            // 로그 처리
            filterChain.doFilter(request, response);
        } finally {
            // MDC 값 제거
            MDC.clear();
        }
    }
}
```

## Nginx와 WAS에서 동일한 식별자 사용하기

일반적으로 서버를 구성할 때 Nginx와 같은 웹 서버를 앞단에 두고 WAS로 요청을 전송하는 구조를 많이 사용한다.

때문에 Nginx 로그와 WAS 로그를 함께 봐야하는 경우가 생기기 마련인데, 이때 Nginx와 WAS가 동일한 request_id를 공유하면 로그 파악에 매우 용이해진다. Nginx는 1.11.0+ 부터 요청에 대한 uid(Unique ID)를 제공하여 요청 구분이 가능하다.

 - `nginx.conf`
    - Nginx의 고유 식별자로 $request_id를 프록시 헤더로 넘긴다.
```conf
server {
    listen 80;
    add_header X-Request-ID $request_id;
    location / {
        proxy_pass http://app_server;
        proxy_set_header X-Request-ID $request_id; # Pass to app server
    }
}
```

 - `MDCFilter`
    - 요청 헤더에 "X-Request-ID"를 읽어서 MDC(로컬 쓰레드)에 저장한다.
```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
class MDCFilter implements Filter {

    @Override
    public void doFilter(final ServletRequest servletRequest, final ServletResponse servletResponse, final FilterChain filterChain) throws IOException, ServletException {
        String requestId = ((HttpServletRequest)servletRequest).getHeader("X-Request-ID");
        MDC.put("request_id", StringUtils.defaultString(requestId, UUID.randomUUID().toString().replaceAll("-", "")));
        filterChain.doFilter(servletRequest, servletResponse);
        MDC.clear();
    }
}
```
