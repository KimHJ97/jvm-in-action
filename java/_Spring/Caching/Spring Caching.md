
# Spring Caching

Spring Cache는 애플리케이션의 성능을 향상시키기 위해 자주 사용되는 데이터를 메모리에 저장하여 재사용할 수 있도록 하는 캐싱 기능을 제공합니다. Spring Cache는 캐싱 추상화를 제공하므로 다양한 캐시 구현체(e.g., Caffeine, Ehcache, Redis, Hazelcast)를 손쉽게 통합할 수 있습니다.

 - 공식 문서: https://docs.spring.io/spring-boot/reference/io/caching.html

## 기본 사용법

 - `의존성 추가`
```groovy
// Spring Caching
implementation 'org.springframework.boot:spring-boot-starter-cache'

// Redis
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
```

 - `캐시 활성화`
    - Spring Cache를 활성화하려면 @EnableCaching 애노테이션을 추가합니다. 보통 @Configuration 클래스에 선언합니다.
    - Cache Storage
        - SimpleCacheManager: 캐시를 직접 등록하여 사용하기 위한 캐시
        - ConcurrentMapCacheManager: JDK의 ConcurrentMap을 기반으로 한 캐시
        - EhCacheCacheManager: EhCache 기반 캐시
        - RedisCacheManager: Redis 기반 캐시
        - CaffeineCacheManager: Caffeine 기반 캐시
        - CompositeCacheManager: 1개 이상의 캐시 매니저를 사용하도록 지원해주는 Mixed 캐시
```java
@Configuration
@EnableCaching // 캐시 활성화
public class CacheConfig {
    
    // CacheManager 직접 등록(로컬의 ConcurrentMap 사용)
    public CacheManager cacheManager() {
         SimpleCacheManager cacheManager = new SimpleCacheManager();
         cacheManager.setCaches(Arrays.asList(new ConcurrentMapCache("default")));
         return cacheManager;
    }
}
```

 - `@Cacheable(캐싱 조회)`
    - 메서드의 실행 결과를 캐시에 저장합니다. 캐시에 데이터가 존재하면 메서드를 실행하지 않고 캐시 데이터를 반환합니다.
    - value: 캐시 이름을 지정합니다.
    - key: 캐시 키를 지정합니다. 기본값은 메서드의 모든 파라미터를 기반으로 키를 생성합니다.
```java
@Cacheable(value = "products", key = "#id")
public Product getProductById(Long id) {
    return productRepository.findById(id).orElseThrow();
}
```

 - `@CachePut(캐시 갱신)`
    - 메서드를 실행하고 항상 결과를 캐시에 저장합니다. 주로 데이터를 갱신할 때 사용합니다.
```java
@CachePut(value = "products", key = "#product.id")
public Product updateProduct(Product product) {
    return productRepository.save(product);
}
```

 - `@CacheEvict(캐시 삭제)`
    - 캐시에 저장된 데이터를 제거할 때 사용합니다.
    - allEntries = true: 해당 캐시의 모든 데이터를 삭제합니다.
    - beforeInvocation = true: 메서드 실행 전에 캐시를 비웁니다.
```java
@CacheEvict(value = "products", key = "#id")
public void deleteProduct(Long id) {
    productRepository.deleteById(id);
}
```

 - `@Caching`
    - 여러 캐싱 애노테이션을 조합하여 사용할 때 유용합니다.
```java
@Caching(
    evict = { @CacheEvict(value = "products", key = "#product.id") },
    put = { @CachePut(value = "products", key = "#product.id") }
)
public Product saveProduct(Product product) {
    return productRepository.save(product);
}
```

## 기본 예제

 - `CacheConfig`
```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        SimpleCacheManager cacheManager = new SimpleCacheManager();
        cacheManager.setCaches(List.of(new ConcurrentMapCache("user")));
        cacheManager.initializeCaches();
        return cacheManager;
    }
}
```

 - `UserService` 
```java
@RequiredArgsConstructor
@Service
public class UserService {

    private final UserRepository userRepository;

    // 정보 조회시 캐시 조회(+DB조회)
    @Cacheable(value = "user", key = "#userId")
    public User find(int userId) {
        log.info("#find: " + userId);
        return userRepository.findById(userId).get().toDomain();
    }

    // 정보 등록시 캐시 갱신
    @CachePut(value = "user", key = "#user.userId")
    public User register(User user) {
        log.info("#register: " + user.getUserId());
        userRepository.save(new UserJpo(user));
        return user;
    }

    // 정보 수정시 캐시 갱신
    @CachePut(value = "user", key = "#userId")
    public User modify(int userId, UserUdo userUdo) {
        log.info("#modify: " + userId);
        User user = new User(userId, userUdo.getName());
        userRepository.save(new UserJpo(user));
        return user;
    }

    // 정보 삭제시 캐시 삭제
    @CacheEvict(value = "user", key = "#userId")
    public void remove(int userId) {
        log.info("#remove: " + userId);
        userRepository.deleteById(userId);
    }
}
```
