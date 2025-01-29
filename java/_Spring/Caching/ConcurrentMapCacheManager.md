# ConcurrentMapCacheManager

ConcurrentMapCacheManager는 Spring Boot에서 기본적으로 제공하는 메모리 기반 캐시 매니저입니다.

내부적으로 Java의 ConcurrentHashMap을 사용하여 캐시 데이터를 저장하며, 싱글 인스턴스 애플리케이션에서 빠르고 간단하게 사용할 수 있습니다.

 - ✅ Spring Boot 기본 캐시 매니저 (추가 설정 없이 사용 가능)
 - ✅ 로컬 메모리 캐시 (별도 저장소 필요 없음)
 - ✅ Thread-safe (ConcurrentHashMap 기반)
 - ✅ TTL(Time-to-Live) 미지원 (데이터 삭제 설정 없음)
 - ✅ 분산 환경 미지원 (서버 간 캐시 공유 불가)

## 장단점

 - ✅ 장점
    - Spring Boot 기본 캐시 매니저 (추가 설정 없이 사용 가능)
    - 빠른 메모리 기반 캐시 (Java ConcurrentHashMap 사용)
    - 멀티 스레드 환경에서도 안전 (Thread-safe)
    - 로컬 캐시로 사용 시 성능 우수
 - ❌ 단점 (제한 사항)
    - TTL(Time-to-Live) 지원 안 함
        - 데이터가 자동으로 만료되지 않음
        - 수동으로 @CacheEvict을 호출해야 함
    - 분산 환경 미지원
        - 서버 간 캐시 공유 불가능
        - 클러스터링 환경에서는 Redis, Ehcache, Caffeine과 같은 분산 캐시 사용 권장
    - 캐시 크기 제한 없음
        - 무한정 ConcurrentHashMap에 저장되므로 메모리 부족 위험 있음
        - EHCache, Caffeine과 달리 최대 크기 제한 기능 없음

## ConcurrentMapCacheManager 설정 및 사용법

 - `의존성 추가`
```java
implementation 'org.springframework.boot:spring-boot-starter-cache'
```

 - `@EnableCaching 활성화`
    - @EnableCaching: Spring Cache 기능을 활성화
    - ConcurrentMapCacheManager("products", "users"): products, users 두 개의 캐시를 관리
```java
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.CacheManager;
import org.springframework.cache.concurrent.ConcurrentMapCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        return new ConcurrentMapCacheManager("products", "users");
    }
}
```

 - `캐싱 적용`
```java
import org.springframework.cache.annotation.Cacheable;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.stereotype.Service;
import java.util.HashMap;
import java.util.Map;

@Service
public class ProductService {

    private final Map<Long, String> productRepository = new HashMap<>();

    // 캐싱 적용 - 캐시에 데이터가 있으면 DB 조회 없이 캐시 데이터를 반환
    @Cacheable(value = "products", key = "#id")
    public String getProductById(Long id) {
        System.out.println("DB 조회: " + id);
        return productRepository.get(id);
    }

    // 캐시 갱신 - 실행된 후 캐시 데이터 업데이트
    @CachePut(value = "products", key = "#id")
    public String updateProduct(Long id, String name) {
        productRepository.put(id, name);
        return name;
    }

    // 캐시 삭제 - 특정 데이터 제거
    @CacheEvict(value = "products", key = "#id")
    public void deleteProduct(Long id) {
        productRepository.remove(id);
    }

    // 모든 캐시 삭제
    @CacheEvict(value = "products", allEntries = true)
    public void clearCache() {
        System.out.println("모든 캐시 삭제");
    }
}
```
