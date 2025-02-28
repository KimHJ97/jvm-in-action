# Filter Logging

## 로깅 대상 정하기

로깅은 모든 엔드포인트에 대해서 처리해야 되고, 요청(Request)과 응답(Response) 내용 등을 로그에 남길 수 있다. 이러한 특징으로 Controller, AOP, Interceptor, Filter 등에서 로깅할 수 있다.

Filter는 서블릿에서 제공하는 기능으로 디스패치 서블릿이 동작하기 전에 동작한다. 즉, Filter 단계에서 가장 먼저 요청(Request) 내용을 받고, 가장 마지막에 응답(Response) 내용을 처리할 수 있다.

 - Controller에서 공통적으로 호출하기
 - AOP
 - Interceptor
 - Filter
```
HTTP 요청 → WAS → 필터 → 디스패처 서블릿 → 인터셉터 → 컨트롤러
```

## 로깅 내용 정하기

 - __요청 로깅__
    - HTTP Method: GET, POST, PUT, PATCH, DELETE 등
    - Request URI: 요청 URL
    - Client IP: 요청 클라이언트 IP
    - Headers: 요청 헤더(Header)
    - Request Param: 요청 URI에 포함되어 있는 Query String
    - Request Body: 요청 본문(Body)
 - __응답 로깅__
    - HTTP Status Code: HTTP 응답 코드(200, 400, 401, 500 등)
    - Response Time: 응답 시간
    - Response Body: 응답 본문(Body)
 - __컨텍스트 정보(MDC 활용)__
    - Request ID: 요청 단위로 로그를 추적할 수 있도록 고유한 식별자
    - User ID: 특정 사용자와 관련된 로그 추적
    - Session ID: 세션과 관련된 로그를 추적
    - Transaction ID: 트랜잭션 단위로 로그를 추적

## Filter

```java
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.util.Enumeration;

import org.springframework.web.filter.OncePerRequestFilter;
import org.springframework.web.util.ContentCachingRequestWrapper;
import org.springframework.web.util.ContentCachingResponseWrapper;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class LogFilter extends OncePerRequestFilter {

	@Override
	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
		throws ServletException, IOException {

		ContentCachingRequestWrapper requestWrapper = new ContentCachingRequestWrapper(request);
		ContentCachingResponseWrapper responseWrapper = new ContentCachingResponseWrapper(response);

		// 필터 체인
		filterChain.doFilter(requestWrapper, response);

		// 요청 및 응답 로깅
		logRequest(requestWrapper);
		logResponse(responseWrapper);
		
		responseWrapper.copyBodyToResponse();
	}

	/**
	 * 요청 로깅
	 * @param request
	 */
	private void logRequest(ContentCachingRequestWrapper request) {
		StringBuilder logMessage = new StringBuilder();

		// 요청 URL 및 HTTP 메서드 로깅
		logMessage.append("\n--- Incoming Request ---\n")
			.append("Method: ").append(request.getMethod()).append("\n")
			.append("URL: ").append(request.getRequestURI()).append("\n");

		// 요청 헤더 로깅
		logMessage.append("Headers:\n");
		Enumeration<String> headerNames = request.getHeaderNames();
		while (headerNames.hasMoreElements()) {
			String headerName = headerNames.nextElement();
			logMessage.append("  ").append(headerName).append(": ").append(request.getHeader(headerName)).append("\n");
		}

		// 요청 바디 로깅 (POST, PUT, PATCH 요청 시만 출력)
		if ("POST".equalsIgnoreCase(request.getMethod()) ||
			"PUT".equalsIgnoreCase(request.getMethod()) ||
			"PATCH".equalsIgnoreCase(request.getMethod())) {
			logMessage.append("Body:\n").append(getRequestBody(request)).append("\n");
		}

		logMessage.append("------------------------");

		log.info(logMessage.toString());
	}

	private String getRequestBody(ContentCachingRequestWrapper request) {
		byte[] content = request.getContentAsByteArray();
		log.info("======================== content length: {}", content.length);
		return content.length > 0 ? new String(content, StandardCharsets.UTF_8) : "[EMPTY]";
	}

	/**
	 * 응답 로깅
	 * @param response
	 */
	private void logResponse(ContentCachingResponseWrapper response) {
		String responseBody = getResponseBody(response);

		log.info("\n--- Outgoing Response ---\nStatus: {}\nBody:\n{}\n------------------------",
			response.getStatus(), responseBody);
	}

	private String getResponseBody(ContentCachingResponseWrapper response) {
		byte[] content = response.getContentAsByteArray();
		return content.length > 0 ? new String(content, StandardCharsets.UTF_8) : "[EMPTY]";
	}

}
```

 - `필터 등록`
```java
@Configuration
public class LogConfig {

	@Bean
	public FilterRegistrationBean<LogFilter> loggingFilter() {
		FilterRegistrationBean<LogFilter> registrationBean = new FilterRegistrationBean<>();
		registrationBean.setFilter(new LogFilter());
		registrationBean.addUrlPatterns("/*");
		registrationBean.setOrder(1);
		return registrationBean;
	}
}
```

## CustomCachingRequestWrapper

Spring의 기본 ContentCachingRequestWrapper는 doFilter() 실행 후에야 요청 본문을 캐싱하므로, doFilter() 이전에는 getContentAsByteArray()를 호출해도 빈 값을 반환하는 문제가 있다.

커스텀 ContentCachingRequestWrapper를 만들어 요청 본문을 즉시 캐싱하도록 커스텀 Wrapper를 만들어사용한다.

 - `ImmediateCachingRequestWrapper`
```java
import jakarta.servlet.ReadListener;
import jakarta.servlet.ServletInputStream;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletRequestWrapper;
import java.io.BufferedReader;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.nio.charset.StandardCharsets;

public class ImmediateCachingRequestWrapper extends HttpServletRequestWrapper {

	private final byte[] cachedBody;

	public ImmediateCachingRequestWrapper(HttpServletRequest request) throws IOException {
		super(request);
		//cachedBody = request.getInputStream().readAllBytes(); // InputStream을 한 번 읽고 캐싱
		cachedBody = readInputStream(request.getInputStream()); // 버퍼 기반으로 InputStream을 읽어 효율적으로 요청 본문을 캐싱
	}

	/**
	 * 요청 InputStream을 효율적으로 읽어 바이트 배열로 변환
	 */
	private byte[] readInputStream(InputStream inputStream) throws IOException {
		ByteArrayOutputStream buffer = new ByteArrayOutputStream();
		byte[] tempBuffer = new byte[8192]; // 8KB 버퍼
		int bytesRead;

		while ((bytesRead = inputStream.read(tempBuffer)) != -1) {
			buffer.write(tempBuffer, 0, bytesRead);
		}
		return buffer.toByteArray();
	}

	@Override
	public ServletInputStream getInputStream() {
		ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(cachedBody);
		return new ServletInputStream() {
			@Override
			public boolean isFinished() {
				return byteArrayInputStream.available() == 0;
			}

			@Override
			public boolean isReady() {
				return true;
			}

			@Override
			public void setReadListener(ReadListener readListener) {
				throw new UnsupportedOperationException();
			}

			@Override
			public int read() {
				return byteArrayInputStream.read();
			}
		};
	}

	@Override
	public BufferedReader getReader() {
		return new BufferedReader(new InputStreamReader(getInputStream(), StandardCharsets.UTF_8));
	}

	/**
	 * 캐싱된 요청 본문 반환
	 */
	public byte[] getCachedBody() {
		return cachedBody;
	}
}
```

 - `LogFilter`
```java
@Slf4j
public class LogFilter extends OncePerRequestFilter {

	@Override
	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
		throws ServletException, IOException {
		ImmediateCachingRequestWrapper requestWrapper = new ImmediateCachingRequestWrapper(request);

		// 요청 로깅
		logRequest(requestWrapper);

		// 필터 체인
		filterChain.doFilter(requestWrapper, response);
	}

	/**
	 * 요청 로깅
	 * @param request
	 */
	private void logRequest(ImmediateCachingRequestWrapper request) {
		StringBuilder logMessage = new StringBuilder();

		// 요청 URL 및 HTTP 메서드 로깅
		logMessage.append("\n--- Incoming Request ---\n")
			.append("Method: ").append(request.getMethod()).append("\n")
			.append("URL: ").append(request.getRequestURI()).append("\n");

		// 요청 헤더 로깅
		logMessage.append("Headers:\n");
		Enumeration<String> headerNames = request.getHeaderNames();
		while (headerNames.hasMoreElements()) {
			String headerName = headerNames.nextElement();
			logMessage.append("  ").append(headerName).append(": ").append(request.getHeader(headerName)).append("\n");
		}

		// 요청 바디 로깅 (POST, PUT, PATCH 요청 시만 출력)
		if ("POST".equalsIgnoreCase(request.getMethod()) ||
			"PUT".equalsIgnoreCase(request.getMethod()) ||
			"PATCH".equalsIgnoreCase(request.getMethod())) {
			logMessage.append("Body:\n").append(getRequestBody(request)).append("\n");
		}

		logMessage.append("------------------------");

		log.info(logMessage.toString());
	}

	private String getRequestBody(ImmediateCachingRequestWrapper request) {
		byte[] content = request.getCachedBody();
		return content.length > 0 ? new String(content, StandardCharsets.UTF_8) : "[EMPTY]";
	}

}
```
