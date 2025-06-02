# Spring Redis 사용법

 - RedisTemplate를 이용한 방법
 - ValueOperations를 이용한 방법
 - @RedisHash를 이용한 방법
 - @Caching 어노테이션을 이용한 방법
 - Redis Client를 직접 이용한 방법(Lettuce, Jedis 등)

<br/>

## 1. 준비

 - `Redis 설치 및 실행(Docker)`
```bash
docker run -d -p 6379:6379 redis
```

 - `의존성 추가`
```groovy
implementation 'org.springframework.boot:spring-boot-starter-data-redis'

```

 - `application.yml`
```yml
spring:
  redis:
    host: localhost
    port: 6379
    password: your_password
    database: 0
```

 - `사용 사례`
    - __캐싱(Caching)__
        - 설명: 자주 액세스하는 데이터를 메모리에 저장해 응답 속도를 높입니다. 데이터베이스, API 호출, 또는 복잡한 계산 결과를 캐싱하여 성능을 최적화합니다
        - 예시: 웹 애플리케이션에서 사용자 세션 데이터, 자주 조회되는 제품 정보, 또는 쿼리 결과를 저장
        - 이유: Redis의 빠른 읽기/쓰기 속도(수십만 OPS)와 TTL(Time-To-Live) 설정으로 캐시 관리에 적합
    - __세션 스토어(Session Store)__
        - 설명: 사용자 세션 데이터를 저장하고 관리합니다. 특히 분산 시스템에서 세션 데이터를 중앙화해 관리
        - 예시: 로그인한 사용자의 세션 정보(예: 토큰, 사용자 ID)를 저장해 웹사이트에서 빠르게 인증
        - 이유: Redis의 빠른 액세스와 데이터 만료 기능이 세션 관리에 효율적
    - __실시간 분석 및 리더보드(Leaderboard)__
        - 설명: 실시간으로 데이터를 집계하거나 순위를 매기는 데 사용. Sorted Set 데이터 구조를 활용해 순위 기반 데이터를 효율적으로 관리
        - 예시: 게임에서 플레이어 점수 순위표, 소셜 미디어의 실시간 좋아요/공유 수 집계
        - 이유: Sorted Set으로 빠르게 순위 계산 및 업데이트 가능
    - __메시지 큐(Message Queue)__
        - 설명: 비동기 작업을 처리하기 위해 작업 큐를 구현. Redis의 List나 Pub/Sub 기능을 사용해 메시지를 전달
        - 예시: 작업 스케줄링(예: 이메일 전송, 데이터 처리), 이벤트 기반 시스템에서 메시지 전달
        - 이유: List의 Push/Pop 연산과 Pub/Sub의 실시간 메시징 기능이 적합
    - __실시간 채팅 및 알림 시스템__
        - 설명: Pub/Sub 기능을 사용해 실시간 메시지 전송 및 알림 구현
        - 예시: 채팅 애플리케이션, 실시간 알림 시스템(예: 새로운 메시지, 이벤트 알림)
        - 이유: Pub/Sub 모델로 구독자들에게 빠르게 데이터 브로드캐스트 가능
    - __지리공간 데이터 처리(Geospatial Data)__
        - 설명: Redis의 Geospatial 인덱스를 사용해 위치 기반 데이터를 저장하고 검색
        - 예시: 근처 음식점 검색, 라이드셰어링 앱에서 가까운 드라이버 매칭
        - 이유: GEO 명령어로 반경 내 데이터 검색이 빠르고 효율적
    - __분산 락(Distributed Lock)__
        - 설명: 분산 시스템에서 동시성 제어를 위해 락을 구현
        - 예시: 여러 서버가 동일한 리소스에 접근할 때 충돌 방지(예: 재고 관리)
        - 이유: Redis의 SETNX(Set if Not Exists) 명령어로 간단히 락 구현 가능

## 1. RedisTemplate를 이용한 방법

RedisTemplate은 Redis의 다양한 데이터 구조(String, List, Hash 등)를 다루기 위해 opsForXxx() 메서드를 제공한다.

 - `RedisTemplate 설정`
    - 기본적으로 Spring Data Redis는 RedisTemplate을 통해 캐시 데이터를 직렬화합니다. 객체를 JSON 형태로 저장하려면 RedisTemplate을 커스터마이징할 수 있습니다.
    - StringRedisSerializer: 캐시 키를 문자열로 직렬화
    - GenericJackson2JsonRedisSerializer: 캐시 값을 JSON으로 직렬화하여 객체 저장
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }

    @Bean
    public RedisCacheConfiguration cacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig()
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));
    }
}
```

 - `RedisRepository`
    - RedisTemplate를 이용해 String, List, Set, Hash, ZSet 등의 데이터 구조를 조작할 수 있다.
    - RedisTemplate은 제네릭 타입(K, V)을 지원하므로 키와 값의 타입을 명시적으로 지정 가능하고, 모든 Redis 작업을 단일 RedisTemplate 객체로 처리하므로 코드가 간결하고 통합적입니다.
    - String: opsForValue()
    - List: opsForList()
    - Set: opsForSet()
    - Hash: opsForHash()
    - ZSet (Sorted Set): opsForZSet()
```java
public class RedisRepository {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    // String 데이터 구조 사용
    public void saveString(String key, String value) {
        redisTemplate.opsForValue().set(key, value); // String 저장
    }

    public String getString(String key) {
        return (String) redisTemplate.opsForValue().get(key); // String 조회
    }

    // Hash 데이터 구조 사용
    public void saveHash(String hashKey, String field, String value) {
        redisTemplate.opsForHash().put(hashKey, field, value); // Hash 저장
    }

    public Object getHash(String hashKey, String field) {
        return redisTemplate.opsForHash().get(hashKey, field); // Hash 조회
    }

    // List 데이터 구조 사용
    public void saveList(String key, String value) {
        redisTemplate.opsForList().rightPush(key, value); // List에 추가
    }
}
```

 - `개인적인 사용 방법`
    - 특정 도메인에 해당하는 XxxRepository를 구현한다.
    - 해당 XxxRepository는 해당 키 Prefix에 대해서만 조작을 한다.
```java
public class MemberSessionRepository {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    // PREFIX를 상수로 정의
    // 여러 도메인에서 동일한 Redis를 사용하는 경우 {domain}.{service}:{key} 처럼 분류한다.
    private final static String MEMBER_SESSION_KEY_PREFIX = "domain.memberSession:";

    public void saveString(String key, String value) {
        redisTemplate.opsForValue().set(MEMBER_SESSION_KEY_PREFIX + key, value);
    }

    public String getString(String key) {
        return redisTemplate.opsForValue().get(MEMBER_SESSION_KEY_PREFIX + key);
    }
}
```
<br/>

## 2. ValueOperations를 이용한 방법

 - `RedisRepository`
    - RedisTemplate에서 opsForXxx()로 반환되는 객체(예: ValueOperations, HashOperations)를 직접 주입받아 사용할 수 있습니다.
    - ValueOperations는 RedisTemplate에서 제공되므로, Spring이 자동으로 주입할 수 있도록 RedisTemplate 빈이 설정되어 있어야 합니다.
    - ValueOperations, HashOperations, ListOperations 등을 직접 주입받아 사용. RedisTemplate에서 opsForXxx()를 호출할 필요 없이 바로 메서드를 호출 가능. 특정 데이터 구조에 특화된 작업에 유용.
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.ValueOperations;

public class RedisRepository {

    @Autowired
    private ValueOperations<String, String> valueOperations; // ValueOperations 주입

    public void saveString(String key, String value) {
        valueOperations.set(key, value); // String 저장
    }

    public String getString(String key) {
        return valueOperations.get(key); // String 조회
    }
}
```

## 3. @RedisHash를 이용한 방법

Spring Data Redis는 JPA와 유사한 방식으로 Redis 데이터를 객체로 매핑하고, 이를 Repository 인터페이스를 통해 관리할 수 있도록 지원합니다.

 - 역할: @RedisHash는 Java 객체를 Redis의 Hash 데이터 구조로 매핑하여 저장하고, Spring Data의 CrudRepository를 통해 CRUD 작업을 수행할 수 있게 합니다. 객체는 Redis Hash로 저장되며, 키는 @RedisHash에 지정된 값과 객체의 ID로 구성됩니다.
 - 한계: @RedisHash는 주로 Hash 데이터 구조에 특화되어 있으며, Redis의 다른 데이터 구조(List, Set, Sorted Set 등)를 직접적으로 매핑하는 기능은 제공하지 않습니다. 또한, 복잡한 쿼리나 Redis의 고급 기능을 활용하기에는 제한적입니다.

<br/>

 - `User & UserRepository`
    - Redis 데이터를 객체로 매핑하여 JPA와 유사한 방식으로 관리
    - @RedisHash로 지정된 클래스는 Redis Hash 구조로 저장됨
    - CrudRepository를 통해 기본적인 CRUD 작업 제공
    - 장점: 객체 지향적인 방식으로 Redis 데이터를 관리. 복잡한 쿼리 없이 간단한 CRUD 작업 가능.
    - 단점: 복잡한 Redis 데이터 구조(List, Set 등)를 다루기에는 제한적. 객체 직렬화/역직렬화로 인한 오버헤드 발생 가능.
```java
// Redis 데이터 객체
import org.springframework.data.redis.core.RedisHash;
import java.io.Serializable;

@Setter
@Getter
@RedisHash("User") // Redis의 Hash 데이터 구조로 저장
public class User implements Serializable {
    private String id;
    private String name;
}

// Repository 인터페이스
import org.springframework.data.repository.CrudRepository;

public interface UserRepository extends CrudRepository<User, String> {
}
```

 - `사용`
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    public void saveUser(User user) {
        userRepository.save(user); // Redis에 저장
    }

    public User getUser(String id) {
        return userRepository.findById(id).orElse(null); // Redis에서 조회
    }
}
```

## 4. @Caching 어노테이션을 이용한 방법

Spring Boot에서 Redis를 캐시 저장소로 사용할 때 @Cacheable, @CachePut, @CacheEvict와 같은 Spring Cache 어노테이션을 활용하면 간단하고 선언적인 방식으로 캐싱을 구현할 수 있습니다.

 - `Spring Cache 주요 어노테이션`
    - Spring Cache는 애플리케이션의 캐싱 로직을 추상화하여 Redis, Ehcache, Caffeine 등 다양한 캐시 저장소를 지원합니다. Redis를 캐시 저장소로 사용할 경우, Spring Data Redis가 이를 지원하며, Redis의 String 또는 Hash 데이터 구조를 사용해 캐시 데이터를 저장합니다.
    - @Cacheable: 메서드 호출 결과를 캐시에 저장하고, 동일한 키로 호출 시 캐시에서 데이터를 반환.
    - @CachePut: 메서드 실행 결과를 캐시에 강제로 저장(캐시 갱신).
    - @CacheEvict: 캐시에서 데이터를 제거.
    - @Caching: 여러 캐시 어노테이션을 조합하여 복잡한 캐싱 로직 구현.
    - @CacheConfig: 클래스 수준에서 캐시 설정을 정의.

 - `캐시 활성화`
    - Spring Boot 애플리케이션에서 캐싱을 활성화하려면 @EnableCaching 어노테이션을 추가한다.
```java
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableCaching
public class CacheConfig {
}
```

 - `UserService`
    - @CacheConfig: 클래스 수준에서 기본 캐시 이름(users) 설정
    - @Cacheable: getUser와 getAllUsers에서 Redis 캐시 확인 후 DB 조회
    - @CachePut: saveUser와 updateUser에서 DB 저장 후 Redis 캐시 갱신
    - @CacheEvict: deleteUser에서 DB 삭제 후 Redis 캐시 제거
    - @Caching: complexUpdate에서 사용자 캐시 갱신(users)과 전체 사용자 캐시 제거(userList)
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.annotation.CacheConfig;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.cache.annotation.Caching;
import org.springframework.stereotype.Service;

@CacheConfig(cacheNames = "users")
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    // 조회: Redis 캐시 확인 후 DB 조회
    @Cacheable(key = "#userId")
    public User getUser(String userId) {
        System.out.println("Fetching user from DB: " + userId);
        return userRepository.findById(userId)
                .orElseThrow(() -> new RuntimeException("User not found: " + userId));
    }

    // 저장: DB 저장 후 Redis 캐시 갱신
    @CachePut(key = "#user.id")
    public User saveUser(User user) {
        System.out.println("Saving user to DB: " + user.getId());
        return userRepository.save(user);
    }

    // 업데이트: DB 업데이트 후 Redis 캐시 갱신
    @CachePut(key = "#user.id")
    public User updateUser(User user) {
        System.out.println("Updating user in DB: " + user.getId());
        if (!userRepository.existsById(user.getId())) {
            throw new RuntimeException("User not found: " + user.getId());
        }
        return userRepository.save(user);
    }

    // 삭제: DB 삭제 후 Redis 캐시 제거
    @CacheEvict(key = "#userId")
    public void deleteUser(String userId) {
        System.out.println("Deleting user from DB: " + userId);
        if (!userRepository.existsById(userId)) {
            throw new RuntimeException("User not found: " + userId);
        }
        userRepository.deleteById(userId);
    }

    // 복잡한 캐싱: 여러 캐시 작업 조합
    // 사용자 캐시 갱신 및 전체 사용자 캐시 제거
    @Caching(
        put = { @CachePut(key = "#user.id") },
        evict = { @CacheEvict(value = "userList", key = "'allUsers'") }
    )
    public User complexUpdate(User user) {
        System.out.println("Complex update in DB: " + user.getId());
        if (!userRepository.existsById(user.getId())) {
            throw new RuntimeException("User not found: " + user.getId());
        }
        return userRepository.save(user);
    }

    // 전체 사용자 목록 조회 (캐시 사용)
    @Cacheable(value = "userList", key = "'allUsers'")
    public Iterable<User> getAllUsers() {
        System.out.println("Fetching all users from DB");
        return userRepository.findAll();
    }
}
```

## 5. Redis Client를 직접 이용한 방법(Lettuce, Jedis 등)

Spring Data Redis를 사용하지 않고, Lettuce 또는 Jedis 클라이언트를 직접 주입받아 Redis에 접근할 수 있습니다. 이 방식은 Spring의 추상화 없이 저수준 API를 사용합니다.

 - `Lettuce 설정`
    - Lettuce 클라이언트를 설정하여 Redis 연결을 관리합니다. RedisClient와 StatefulRedisConnection을 빈으로 등록합니다.
    - RedisClient: Redis 서버 연결 설정
    - StatefulRedisConnection: Redis 연결 관리
    - RedisCommands: 동기 API로 Redis 명령 실행
```java
import io.lettuce.core.RedisClient;
import io.lettuce.core.api.StatefulRedisConnection;
import io.lettuce.core.api.sync.RedisCommands;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class LettuceConfig {

    @Bean
    public RedisClient redisClient() {
        return RedisClient.create("redis://localhost:6379/0");
    }

    @Bean
    public StatefulRedisConnection<String, String> redisConnection(RedisClient redisClient) {
        return redisClient.connect();
    }

    @Bean
    public RedisCommands<String, String> redisCommands(StatefulRedisConnection<String, String> connection) {
        return connection.sync();
    }
}
```

 - `Lettuce 사용`
```java
import com.fasterxml.jackson.databind.ObjectMapper;
import io.lettuce.core.api.sync.RedisCommands;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;

@RequiredArgsConstructor
@Service
public class UserService {

    private final UserRepository userRepository;
    private final RedisCommands<String, String> redisCommands;
    private final ObjectMapper objectMapper;

    private static final String USER_CACHE_PREFIX = "users:";
    private static final String USER_LIST_KEY = "userList:allUsers";
    private static final long TTL_SECONDS = 600; // 10분 TTL

    // 조회: Redis 캐시 확인 후 DB 조회 (@Cacheable 대체)
    public User getUser(String userId) throws Exception {
        String cacheKey = USER_CACHE_PREFIX + userId;
        String cachedData = redisCommands.get(cacheKey);

        if (cachedData != null) {
            System.out.println("Fetching user from Redis: " + userId);
            return objectMapper.readValue(cachedData, User.class);
        }

        System.out.println("Fetching user from DB: " + userId);
        User user = userRepository.findById(userId)
                .orElseThrow(() -> new RuntimeException("User not found: " + userId));
        
        // Redis에 캐시 저장 (TTL 설정)
        redisCommands.setex(cacheKey, TTL_SECONDS, objectMapper.writeValueAsString(user));
        return user;
    }

    // 저장: DB 저장 후 Redis 캐시 갱신 (@CachePut 대체)
    public User saveUser(User user) throws Exception {
        System.out.println("Saving user to DB: " + user.getId());
        User savedUser = userRepository.save(user);
        
        // Redis 캐시 갱신
        String cacheKey = USER_CACHE_PREFIX + user.getId();
        redisCommands.setex(cacheKey, TTL_SECONDS, objectMapper.writeValueAsString(savedUser));
        return savedUser;
    }

    // 업데이트: DB 업데이트 후 Redis 캐시 갱신 (@CachePut 대체)
    public User updateUser(User user) throws Exception {
        System.out.println("Updating user in DB: " + user.getId());
        if (!userRepository.existsById(user.getId())) {
            throw new RuntimeException("User not found: " + user.getId());
        }
        User updatedUser = userRepository.save(user);
        
        // Redis 캐시 갱신
        String cacheKey = USER_CACHE_PREFIX + user.getId();
        redisCommands.setex(cacheKey, TTL_SECONDS, objectMapper.writeValueAsString(updatedUser));
        return updatedUser;
    }

    // 삭제: DB 삭제 후 Redis 캐시 제거 (@CacheEvict 대체)
    public void deleteUser(String userId) {
        System.out.println("Deleting user from DB: " + userId);
        if (!userRepository.existsById(userId)) {
            throw new RuntimeException("User not found: " + userId);
        }
        userRepository.deleteById(userId);
        
        // Redis 캐시 제거
        redisCommands.del(USER_CACHE_PREFIX + userId);
    }

    // 복잡한 캐싱: 사용자 캐시 갱신 및 전체 사용자 캐시 제거 (@Caching 대체)
    public User complexUpdate(User user) throws Exception {
        System.out.println("Complex update in DB: " + user.getId());
        if (!userRepository.existsById(user.getId())) {
            throw new RuntimeException("User not found: " + user.getId());
        }
        User updatedUser = userRepository.save(user);
        
        // Redis 캐시 갱신 및 전체 사용자 캐시 제거
        String cacheKey = USER_CACHE_PREFIX + user.getId();
        redisCommands.setex(cacheKey, TTL_SECONDS, objectMapper.writeValueAsString(updatedUser));
        redisCommands.del(USER_LIST_KEY);
        return updatedUser;
    }

    // 전체 사용자 목록 조회: Redis List 사용 (@Cacheable 대체)
    public List<User> getAllUsers() throws Exception {
        String cachedData = redisCommands.get(USER_LIST_KEY);
        if (cachedData != null) {
            System.out.println("Fetching all users from Redis");
            return objectMapper.readValue(cachedData, objectMapper.getTypeFactory().constructCollectionType(List.class, User.class));
        }

        System.out.println("Fetching all users from DB");
        List<User> users = userRepository.findAll();
        redisCommands.setex(USER_LIST_KEY, TTL_SECONDS, objectMapper.writeValueAsString(users));
        return users;
    }
}
```
