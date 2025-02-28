# RequestBody

@RequestBody는 Spring MVC에서 클라이언트가 보낸 HTTP 요청의 본문(body)을 Java 객체로 변환할 때 사용하는 어노테이션입니다. 주로 POST, PUT, PATCH 요청에서 JSON, XML 등의 데이터를 받아올 때 사용됩니다.

 - 내부적으로 HttpMessageConverter를 사용하여 JSON, XML 등의 데이터를 Java 객체로 변환함
 - 기본적으로 MappingJackson2HttpMessageConverter가 사용되며, Jackson 라이브러리가 필요함

```java
// DTO 정의
@Getter
@NoArgsConstructor
public class UserCreateRequest {
    private String name;
    private int age;
}

// Controller 정의
@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public ResponseEntity<String> createUser(@RequestBody UserCreateRequest request) {
        return ResponseEntity.ok("User created: " + request.getName());
    }
}
```
