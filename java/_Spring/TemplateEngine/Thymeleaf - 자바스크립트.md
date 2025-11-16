# Thymeleaf 자바스크립트 인라인

타임리프는 자바스크립트에서 타임리프를 편리하게 사용할 수 있는 자바스크립트 인라인 기능을 제공합니다.
 - 타임리프의 자바스크립트 인라인 기능을 이용하면 Javascript에 변수를 담거나, 객체를 담는 경우 쉽게 사용이 가능합니다.
 - 문자열은 쌍따옴표(")을 자동으로 추가하여 대입해주고, 객체인 경우 JSON 형태로 변환하여 대입해준다.
 - 또한, 변수에 포함된 쌍따옴표(") 같은 Javascript에 문제가 될 수 있는 문자가 있으면 이스케이프(escape) 처리도 자동으로 해줍니다. (ex: " -> \")

```html
<script th:inline="javascript">
    // 리터럴 주입
    var username = "[[${user.username}]]"; // 쌍따옴표를 이용하여 변수를 넣어주어야 한다.

    // 객체 주입: JSON 문자열로 만들어서 객체에 대입하거나, 프로퍼티를 하나씩 대입
    var user = {
        username: '[[${user.username}]]',
        age: [[${user.age}]]
    }

    // 반복문
    [# th:each="user, stat : ${users}"]
    var user[[${stat.count}]] = [[${user}]];
    [/]
</script>
```
<br/>

## 내추럴 템플릿

```html
<script th:inline="javascript">
    var username = /*[[${user.username}]]*/ "USER_NAME";
</script>
```
