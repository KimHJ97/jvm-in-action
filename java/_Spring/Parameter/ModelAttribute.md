# ModelAttribute

## 메서드 레벨

컨트롤러 클래스 내에서 @ModelAttribute를 메서드에 붙이면, 해당 메서드가 모든 요청을 처리하기 전에 실행되어 모델에 데이터를 추가합니다. 이는 공통 데이터를 뷰에 제공할 때 유용합니다.

```java
@Controller
public class UserController {

    @ModelAttribute("siteName")
    public String addSiteName() {
        return "My Awesome Website"; // 모든 뷰에서 "siteName"이라는 속성 사용 가능
    }

    @GetMapping("/home")
    public String home(Model model) {
        // "siteName"이 자동으로 모델에 추가됨
        return "home";
    }
}
```

## 파라미터 레벨

 - @AllArgsConstructor
 - @NoArgsConstructor + @Setter
 - @NoArgsConstructor와 @AllArgsConstructor가 있는 경우 @NoArgsConstructor가 우선순위가 높다. 즉, @NoArgsConstructor가 있으면 @Setter가 필수적이다.
 - URL 쿼리 스트링, x-www-form-urlencoded, form-data 처리
```java
@AllArgsConstructor
public class TestRequest {
    private String name;
    private int age;
}

@Setter
@NoArgsConstructor
public class TestRequest2 {
    private String name;
    private int age;
}
```
