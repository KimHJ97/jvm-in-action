# Thymeleaf 기본

 - https://www.thymeleaf.org/


## 1. 기본 설정

 - `의존성 추가`
```groovy
// Spring Legacy
implementation 'org.thymeleaf:thymeleaf:3.1.0.RELEASE'
implementation 'org.thymeleaf:thymeleaf-spring5:3.1.0.RELEASE'
implementation 'nz.net.ultraq.thymeleaf:thymeleaf-layout-dialect:3.1.0'

// Spring Boot
implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
```
<br/>

 - `Thymeleaf 설정 - Java Config`
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.support.ResourceBundleMessageSource;
import org.springframework.web.servlet.ViewResolver;
import org.thymeleaf.spring5.SpringTemplateEngine;
import org.thymeleaf.spring5.templateresolver.SpringResourceTemplateResolver;
import org.thymeleaf.spring5.view.ThymeleafViewResolver;
import org.thymeleaf.templatemode.TemplateMode;
import nz.net.ultraq.thymeleaf.LayoutDialect;

@Configuration
public class ThymeleafConfig {

	@Bean
	public SpringResourceTemplateResolver templateResolver() {
		SpringResourceTemplateResolver  templateResolver = new SpringResourceTemplateResolver();
		templateResolver.setPrefix("/WEB-INF/views/");
		templateResolver.setSuffix(".html");
		templateResolver.setTemplateMode(TemplateMode.HTML);
		templateResolver.setCharacterEncoding("UTF-8");
		templateResolver.setCacheable(false); // 캐시 사용 X(사용시 html 수정시 서버 재기동)
		return templateResolver;
	}

	@Bean
	public LayoutDialect layoutDialect() {
		return new LayoutDialect();
	}

	@Bean
	public ResourceBundleMessageSource messageSource() {
		ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
		messageSource.setDefaultEncoding("utf-8");
		messageSource.setBasename("messages"); // messages.properties
		return messageSource;
	}

	@Bean
	public SpringTemplateEngine templateEngine() {
		SpringTemplateEngine templateEngine = new SpringTemplateEngine();
		templateEngine.setTemplateResolver(templateResolver());
		templateEngine.setEnableSpringELCompiler(true); // Spring EL 사용
		templateEngine.addDialect(layoutDialect());
		templateEngine.setTemplateEngineMessageSource(messageSource()); // property 파일의 값(메세지)을 사용할 경우
		// templateEngine.addDialect(new SpringSecurityDialect());

		return templateEngine;
	}

	@Bean
	public ViewResolver thymeleafViewResolver() {
		ThymeleafViewResolver viewResolver = new ThymeleafViewResolver();
		viewResolver.setTemplateEngine(templateEngine());
		viewResolver.setCharacterEncoding("UTF-8");
		viewResolver.setOrder(1);
		return viewResolver;
	}

}
```
<br/>

 - `Thymeleaf 설정 - application.yml`
    - Spring Boot의 경우 thymeleaf 의존성을 추가한 것만으로도 ViewResolver에게 반환시 기본적으로 templates 폴더 하위에 *.html 파일과 매칭되도록 되어있다.
```yml
# application.properties 파일
# thymeleaf 사용 여부
spring.thymeleaf.enabled=true
# template 경로 접두사
spring.thymeleaf.prefix=classpath:/templates/
# template 경로 접미사
spring.thymeleaf.suffix=.html
# cache 활성화 여부, 개발환경에서는 비 활성화
spring.thymeleaf.cache=true;
# template 인코딩
spring.thymeleaf.encoding=UTF-8
#기본 template 모드, TemplateMode에 정의 (HTML, XML, TEXT, JAVASCRIPT 등)
spring.thymeleaf.mode=HTML
# 렌더링 전에 template 존재 여부 확인 
spring.thymeleaf.check-template=true
# template 위치 존재 여부 확인
spring.thymeleaf.check-template-location=true
```
<br/>

## 2. 기본 문법

### 2-1. 표현식

 - `표현식`
```
변수 표현식 -> ${}
선택 변수 표현식 -> *{}
메시지 표현식 -> #{}
링크 URL 표현식 -> @{}
조각 표현식 -> ~{}
```

 - `표현식 예시`
    - `th:text`와 `[[]]` 표현식은 이스케이프를 제공한다. HTML 태그 내용의 경우 `&lt;`, `&gt;`와 같이 치환된다.
    - `th:utext`와 `[()]` 표현식을 사용하면 언이스케이프하게 사용할 수 있다.
```html
<span th:text="${data}">
<span th:utext="${data}">
<span>hello [[${data}]]</span>
<span>hello [(${data})]</span>
```
<br/>

### 2-2. 변수

 - `변수`
```
단순 변수
 - data

Object
 - user.username : user의 username을 프로퍼티 접근 user.getUsername()
 - user['username'] : 위와 같음 user.getUsername()
 - user.getUsername() : user의 getUsername() 을 직접 호출

List
 - users[0].username : List에서 첫 번째 회원을 찾고 username 프로퍼티 접근
 - users[0]['username'] : 위와 같음
 - users[0].getUsername() : List에서 첫 번째 회원을 찾고 메서드 직접 호출
 - list.get(0).getUsername(): List의 get 메서드를 통해 데이터를 찾아 username 프로퍼티 접근

Map
 - userMap['userA'].username : Map에서 userA를 찾고, username 프로퍼티 접근
 - userMap['userA']['username'] : 위와 같음
 - userMap['userA'].getUsername() : Map에서 userA를 찾고 메서드 직접 호출
 - map.get("userA").getUsername(): Map에서 get 메서드를 통해 userA 데이터를 찾아 username 프로퍼티 접근
```
<br/>

### 2-3. 반복 & 조건

 - `반복`
```html
<tr th:each="item : ${memberList}">
    <td th:text="${item.name}">이름</td>
    <td th:text="${item.age}">나이</td>
    <td th:text="${item.gender}">성별</td>
</tr>
```
<br/>

 - `조건`
```html
<!-- if, unless -->
<span th:text="'미성년자'" th:if="${member.age lt 20}"></span>
<span th:text="'미성년자'" th:unless="${member.age ge 20}"></span>

<!-- switch-case -->
<div th:switch="${member.gender}">
    <span th:case="MALE">남성</span>
    <span th:case="FEMALE">여성</span>
    <span th:case="*">미확인</span>	
</div>
```