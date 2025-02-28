# PathVariable

@PathVariable은 Spring MVC에서 URL 경로(path)에서 값을 추출하여 컨트롤러 메서드의 매개변수로 바인딩할 때 사용하는 어노테이션입니다.

RESTful API에서 자원을 식별하는 데 자주 사용됩니다.

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    /**
     * 단일 값 추출
     * GET /api/users/100
     **/
    @GetMapping("/{userId}")
    public ResponseEntity<String> getUser(@PathVariable Long userId) {
        return ResponseEntity.ok("User ID: " + userId);
    }

    /**
     * 단일 값 추출(변수명이 다른 경우)
     * GET /api/users/100
     **/
    @GetMapping("/{id}")
    public ResponseEntity<String> getUser(@PathVariable("id") Long userId) {
        return ResponseEntity.ok("User ID: " + userId);
    }


    /**
     * 다중 값 추출
     * GET /api/users/100/orders/200
     * 
     **/
    @GetMapping("/{userId}/orders/{orderId}")
    public ResponseEntity<String> getUserOrder(
            @PathVariable Long userId,
            @PathVariable Long orderId) {
        return ResponseEntity.ok("User ID: " + userId + ", Order ID: " + orderId);
    }

}
```
