# RequestParam

@RequestParam은 Spring MVC에서 GET, POST 등의 HTTP 요청에서 쿼리 파라미터(query parameter)나 폼 데이터(form-data)를 컨트롤러 메서드의 매개변수로 바인딩할 때 사용하는 어노테이션입니다.

```java
// 쿼리 파라미터 & 폼 데이터 받기
@RestController
@RequestMapping("/api/users")
public class UserController {

    // GET /api/users?name=홍길동
    @GetMapping
    public ResponseEntity<String> getUser(@RequestParam(defaultValue = "Guest") String name) {
        return ResponseEntity.ok("User name: " + name);
    }

    // GET /api/users/hobbies?hobby=축구&hobby=농구&hobby=독서
    @GetMapping("/hobbies")
    public ResponseEntity<String> getHobbies(@RequestParam List<String> hobby) {
        return ResponseEntity.ok("Hobbies: " + hobby);
    }
}

// 파일 받기
@RestController
@RequestMapping("/api/files")
public class FileController {

    /**
     * 파일 받기
     * POST /api/files/upload
     * Content-Type: multipart/form-data
     **/
    @PostMapping("/upload")
    public ResponseEntity<String> uploadFile(@RequestParam("file") MultipartFile file) {
        if (file.isEmpty()) {
            return ResponseEntity.badRequest().body("파일이 없습니다.");
        }
        return ResponseEntity.ok("파일 업로드 성공: " + file.getOriginalFilename());
    }


    /**
     * 파일 여러개 받기
     * POST /api/files/uploads
     * Content-Type: multipart/form-data
     * files=file1.jpg
     * files=file2.png
     **/
    @PostMapping("/uploads")
    public ResponseEntity<String> uploadMultipleFiles(@RequestParam("files") List<MultipartFile> files) {
        if (files.isEmpty()) {
            return ResponseEntity.badRequest().body("파일이 없습니다.");
        }

        List<String> fileNames = files.stream()
                .map(MultipartFile::getOriginalFilename)
                .collect(Collectors.toList());

        return ResponseEntity.ok("업로드된 파일 목록: " + fileNames);
    }

    /**
     * 파일 + 텍스트 데이터 동시에 받기
     * POST /api/files/uploadWithData
     * Content-Type: multipart/form-data
     * file=file1.jpg
     * username=홍길동
     **/
    @PostMapping("/uploadWithData")
    public ResponseEntity<String> uploadFileWithData(
            @RequestParam("file") MultipartFile file,
            @RequestParam("username") String username) {

        if (file.isEmpty()) {
            return ResponseEntity.badRequest().body("파일이 없습니다.");
        }

        return ResponseEntity.ok(username + "님의 파일: " + file.getOriginalFilename());
    }

}
```
