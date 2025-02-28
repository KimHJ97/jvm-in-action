# RequestPart

@RequestPart는 Spring MVC에서 multipart/form-data 요청을 처리할 때 사용되는 어노테이션입니다.
 - @RequestParam과 비슷하지만, 파일과 JSON 데이터를 함께 받을 때 유용합니다.
 - 파일(MultipartFile)과 객체(DTO)를 동시에 받을 때 사용합니다.
 - @RequestBody처럼 JSON 데이터를 받을 수 있지만, multipart/form-data와 함께 사용됩니다.

```java
@Getter
@NoArgsConstructor
public class FileMetadata {
    private String description;
}

@RestController
@RequestMapping("/api/files")
public class FileController {

    /**
     * POST /api/files/upload
     * Content-Type: multipart/form-data
     * file=file1.jpg
     **/
    @PostMapping("/upload")
    public ResponseEntity<String> uploadFile(@RequestPart("file") MultipartFile file) {
        return ResponseEntity.ok("파일 업로드 성공: " + file.getOriginalFilename());
    }

    /**
     * 파일 + JSON 데이터 함께 받기
     * POST /api/files/uploadWithData
     * Content-Type: multipart/form-data
     * file=file1.jpg
     * metadata={ "description": "프로필 사진" }
     **/
    @PostMapping("/uploadWithData")
    public ResponseEntity<String> uploadFileWithData(
            @RequestPart("file") MultipartFile file,
            @RequestPart("metadata") FileMetadata metadata) {

        return ResponseEntity.ok("파일 업로드 성공: " + file.getOriginalFilename() +
                ", 설명: " + metadata.getDescription());
    }

    /**
     * 여러 개의 파일 업로드
     * POST /api/files/multipleFiles
     * Content-Type: multipart/form-data
     * files=file1.jpg
     * files=file2.png
     **/
    @PostMapping("/multipleFiles")
    public ResponseEntity<String> uploadMultipleFiles(@RequestPart("files") List<MultipartFile> files) {
        List<String> fileNames = files.stream()
                .map(MultipartFile::getOriginalFilename)
                .collect(Collectors.toList());

        return ResponseEntity.ok("업로드된 파일 목록: " + fileNames);
    }

    /**
     * 여러 개의 파일 + JSON 데이터 함께 받기
     * POST /api/files/multipleFilesWithData
     * Content-Type: multipart/form-data
     * files=file1.jpg
     * files=file2.png
     * metadata={ "description": "프로필 및 커버 사진" }
     **/
    @PostMapping("/multipleFilesWithData")
    public ResponseEntity<String> uploadMultipleFilesWithData(
            @RequestPart("files") List<MultipartFile> files,
            @RequestPart("metadata") FileMetadata metadata) {

        List<String> fileNames = files.stream()
                .map(MultipartFile::getOriginalFilename)
                .collect(Collectors.toList());

        return ResponseEntity.ok("업로드된 파일: " + fileNames + ", 설명: " + metadata.getDescription());
    }

}
```
