# Thymeleaf 템플릿 레이아웃

 - `의존성 추가`
```groovy
implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
implementation 'nz.net.ultraq.thymeleaf:thymeleaf-layout-dialect:3.2.0'
```

## 1. 템플릿 조각

 - `문법`
    - th:fragment="이름" 속성을 추가하면 템플릿 조각이되어 다른 곳에서 불러와 사용이 가능해진다.
    - th:insert, th:replace로 템플릿 조각을 가져와 불러올 수 있다.
```
th:fragment="조각이름"
 - 해당 태그로 선언된 태그 내부가 템플릿이 되며 속성명이 템플릿 조각 이름이 된다.
 - 해당 템플릿 조각을 사용하고 싶은 다른 영역에서 해당 이름을 사용해 템플릿을 가져올 수 있다.
 - 또한, 파라미터를 전달해줄수도 있는데, 해당 파라미터를 템플릿 조각 내에서 사용할 수 있다.
 - ex) <tag th:fragment="조각 이름"> 내용 </tag>

th:insert="~{템플릿 HTML 경로 :: 조각이름"}
 - 해당 파일에 있는 조각(fragment) 중에 조각이름에 해당하는 템플릿을 가져와 안에 주입한다.
 - ex) <div th:insert="~{template/fragment/footer :: copy}"></div>

th:replace="~{템플릿 HTML 경로 :: 조각이름"}
 - 해당 파일에 있는 각(fragment) 중에 조각이름에 해당하는 템플릿을 가져와 해당 태그와 교체한다.
 - ex) <div th:replace="~{template/fragment/footer :: copy}"></div>
```
<br/>

 - `예시`
```html
<!-- main.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <h1>부분 포함</h1>

    <h2>부분 포함 insert</h2>
    <div th:insert="~{template/fragment/footer :: copy}"></div>

    <h2>부분 포함 replace</h2>
    <div th:replace="~{template/fragment/footer :: copy}"></div>

    <h2>부분 포함 단순 표현식</h2>
    <div th:replace="template/fragment/footer :: copy"></div>

    <h1>파라미터 사용</h1>
    <div th:replace="~{template/fragment/footer :: copyParam ('데이터1', '데이터2')}"></div>
</body>
</html>


<!-- footer.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<body>
    <footer th:fragment="copy">
        푸터 자리 입니다.
    </footer>

    <footer th:fragment="copyParam (param1, param2)">
        <p>파라미터 자리 입니다.</p>
        <p th:text="${param1}"></p>
        <p th:text="${param2}"></p>
    </footer>
</body>
</html>
```
<br/>

## 2. 템플릿 레이아웃

템플릿 조각으로는 JSP의 include처럼 다른 템플릿의 특정 영역을 가져와 사용한다. Thymeleaf는 더 확장하여 코드 조각을 레이아웃 개념으로 사용할 수 있다.

예를 들어, 헤더에 공통으로 사용하는 css, javascript 같은 정보들이 있는데, 이러한 공통 정보들을 한 곳에 모아두고, 공통으로 사용된다. 이때, 각 페이지마다 필요한 정보는 동적으로 사용하고 싶다면 다음과 같이 템플릿 조각(fragment)을 레이아웃처럼 사용할 수 있다.

```html
<!-- base.html -->
<html xmlns:th="http://www.thymeleaf.org">
<head th:fragment="common_header(title,links)">

    <title th:replace="${title}">레이아웃 타이틀</title>

    <!-- 공통 -->
    <link rel="stylesheet" type="text/css" media="all" th:href="@{/css/ awesomeapp.css}">
    <link rel="shortcut icon" th:href="@{/images/favicon.ico}">
    <script type="text/javascript" th:src="@{/sh/scripts/codebase.js}"></script>

    <!-- 추가 -->
    <th:block th:replace="${links}"/>
</head>


<!-- main.html -->
 <!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head th:replace="template/layout/base :: common_header(~{::title},~{::link})">
    <title>메인 타이틀</title>
    <link rel="stylesheet" th:href="@{/css/bootstrap.min.css}">
    <link rel="stylesheet" th:href="@{/themes/smoothness/jquery-ui.css}">
</head>
<body>메인 컨텐츠</body>
</html>
```
<br/>

## 3. 템플릿 레이아웃 플러그인

 - `공통 부분 만들기`
    - 레이아웃에서 사용될 공통 부분(header, footer)을 fragment로 만든다.
```html
<!-- templates/fragments/header.html -->
<html lagn="ko" xmlns:th="http://www.thymeleaf.org">
<div th:fragment="headerFragment">
	<div id="header" class="header navbar-default">
		헤더입니다.
	</div>
</div>
</html>

<!-- templates/fragments/footer.html -->
<html lagn="ko" xmlns:th="http://www.thymeleaf.org">
<div th:fragment="footerFragment">
	<div id="footer" class="footer">
		푸터입니다.
	</div>
</div>
</html>
```
<br/>

 - `레이아웃 만들기`
    - HTML 문서의 전체 디자인 틀을 가지고 있고, 공통 부분을 포함하는 레이아웃을 만든다.
    - 공통 부분은 th:replace로 대체하고, 콘텐츠 부분은 layout:fragment로 불러온다.
```html
<!-- templates/layouts/default.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org" xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout">
<head>
	<meta charset="utf-8" />
	<meta http-equiv="X-UA-Compatible" content="IE=edge">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>사이트 제목</title>

	<link href="/assets/css/default/app.min.css" rel="stylesheet" />
	<script src="/assets/js/app.min.js"></script>
</head>
<body>
	
	<!-- begin #page-container -->
	<div id="page-container" class="page-container fade page-sidebar-fixed page-header-fixed">

		<!-- begin #header -->
		<div th:replace="fragments/header::headerFragment"></div>
		<!-- end #header -->

		<!-- begin #content -->
		<div layout:fragment="content"></div>
		<!-- end #content -->

		<!-- begin #footer -->
		<div th:replace="fragments/footer::footerFragment"></div>
		<!-- begin #footer -->

	</div>
	<!-- end page container -->
</body>
</html>
```
<br/>

 - `콘텐츠 만들기`
    - 내용 부분을 담는 HTML 파일을 만든다.
    - 해당 파일에서 어떤 레이아웃(틀)을 사용할 지 지정할 수 있다.
    - 즉, 지정한 레이아웃 HTML 파일이 노출되며, 공통 부분은 이미 정의되어 있고 내용(Contents) 부분만 현재 파일의 내용으로 대체된다.
```html
<!-- pages/main -->
<!DOCTYPE html>
<html xmlns:th="http//www.thymeleaf.org"
	xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
	layout:decorate="~{layouts/default}">

<div layout:fragment="content">
	<div id="content" class="content">
		콘텐츠 내용
	</div>
</div>

</html>
```



