# Redis

Spring Boot에서 Redis를 캐시로 활용하면 분산 환경에서도 캐시 데이터 유지가 가능하고, 빠른 읽기/쓰기 성능을 제공할 수 있습니다. Redis는 Key-Value Store 방식의 인메모리 데이터베이스로, 캐싱뿐만 아니라 세션 저장, 메시지 큐, 실시간 데이터 처리에도 사용됩니다.

## Redis 설정 및 적용 예제

 - `의존성 추가`
```groovy
implementation 'org.springframework.boot:spring-boot-starter-cache'
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
```

 - `Redis 설정 (application.yml)`
    - spring.cache.type=redis → Spring Boot가 Redis 캐시를 사용하도록 설정
    - spring.redis.host=localhost → Redis 서버 주소
    - spring.redis.port=6379 → Redis 기본 포트
    - spring.redis.timeout=6000ms → Redis 응답 시간 제한
```yml
spring:
  cache:
    type: redis
  redis:
    host: localhost
    port: 6379
    password: ""  # Redis에 비밀번호 설정 시 입력
    timeout: 6000ms
```

 - `CacheManager 설정`
```java
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.CacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import java.time.Duration;

@Configuration
@EnableCaching // 캐싱 기능을 활성화
public class RedisCacheConfig {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
        RedisCacheConfiguration cacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10)) // 캐시 TTL 10분
            .disableCachingNullValues() // null 값은 캐싱하지 않도록 설정
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()
                )
            ); // 객체를 JSON 형태로 직렬화하여 Redis에 저장

        return RedisCacheManager.builder(redisConnectionFactory)
            .cacheDefaults(cacheConfiguration)
            .build();
    }
}
```

 - `캐싱 적용 (@Cacheable, @CachePut, @CacheEvict)`
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

    // 캐싱 적용 - 캐시에 데이터가 있으면 DB 조회를 생략하고 캐시된 데이터를 반환
    @Cacheable(value = "products", key = "#id")
    public String getProductById(Long id) {
        System.out.println("DB 조회: " + id);
        return productRepository.get(id);
    }

    // 캐시 갱신 - 항상 실행되며 결과를 Redis 캐시에 저장
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
