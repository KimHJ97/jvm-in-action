# EHCache

EHCache는 Spring Boot에서 자주 사용되는 강력한 로컬 캐시 솔루션입니다. EHCache는 디스크 저장, TTL(Time-To-Live) 설정, LRU(Least Recently Used) 기반 캐시 관리 등 다양한 기능을 제공합니다.

## EHCache 설정 및 적용 예제

 - `의존성 추가`
    - 최신 EHCache 3.x는 javax.cache(JCache)와 통합되어 별도로 JCache API가 필요합니다. 이 예제에서는 2.x 버전을 사용합니다.
```groovy
implementation 'org.springframework.boot:spring-boot-starter-cache'
implementation 'net.sf.ehcache:ehcache:2.10.9.2'
```

 - `EHCache 설정 파일 작성`
    - Spring Boot는 ehcache.xml 설정 파일을 resources 폴더에 위치시켜야 합니다. (src/main/resources/ehcache.xml)
    - maxEntriesLocalHeap: 캐시에 저장할 최대 항목 개수 (메모리 내).
    - timeToIdleSeconds: 마지막 접근 후 캐시가 만료되는 시간.
    - timeToLiveSeconds: 캐시 항목의 생명 주기 (TTL).
    - memoryStoreEvictionPolicy:
        - LRU (Least Recently Used): 가장 오래 사용되지 않은 항목 제거.
        - LFU (Least Frequently Used): 가장 적게 사용된 항목 제거.
```xml
<?xml version="1.0" encoding="UTF-8"?>
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="http://www.ehcache.org/ehcache.xsd"
         updateCheck="false">

    <!-- 디폴트 캐시 설정 -->
    <defaultCache
        maxEntriesLocalHeap="1000"
        eternal="false"
        timeToIdleSeconds="120"
        timeToLiveSeconds="300"
        memoryStoreEvictionPolicy="LRU"/>

    <!-- 특정 캐시 정의 -->
    <cache name="products"
           maxEntriesLocalHeap="500"
           eternal="false"
           timeToIdleSeconds="60"
           timeToLiveSeconds="300"
           memoryStoreEvictionPolicy="LRU"/>

    <cache name="users"
           maxEntriesLocalHeap="1000"
           eternal="false"
           timeToIdleSeconds="300"
           timeToLiveSeconds="600"
           memoryStoreEvictionPolicy="LFU"/>
</ehcache>
```

 - `Spring에서 EHCache CacheManager 설정`
    - @EnableCaching을 선언하여 캐싱 기능을 활성화합니다.
    - EhCacheCacheManager를 Spring의 CacheManager로 등록합니다.
    - ehcache.xml을 로드하여 EHCache를 초기화합니다.
```java
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.CacheManager;
import org.springframework.cache.ehcache.EhCacheCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import net.sf.ehcache.config.ConfigurationFactory;
import net.sf.ehcache.CacheManager as EhCacheManager;
import net.sf.ehcache.config.Configuration as EhCacheConfiguration;

@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        return new EhCacheCacheManager(ehCacheManager());
    }

    @Bean
    public EhCacheManager ehCacheManager() {
        return EhCacheManager.create(ConfigurationFactory.parseConfiguration(
                getClass().getResource("/ehcache.xml")
        ));
    }
}
```

 - `캐싱 적용`
    - @Cacheable, @CachePut, @CacheEvict을 사용하여 캐싱을 적용할 수 있습니다.
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

    // 캐싱 적용 - 캐시에 데이터가 있으면 DB를 조회하지 않고 캐시에서 반환
    @Cacheable(value = "products", key = "#id")
    public String getProductById(Long id) {
        System.out.println("DB 조회: " + id);
        return productRepository.get(id);
    }

    // 캐시 갱신 - 항상 실행되며 결과를 캐시에 저장
    @CachePut(value = "products", key = "#id")
    public String updateProduct(Long id, String name) {
        productRepository.put(id, name);
        return name;
    }

    // 캐시 삭제 - 특정 항목 제거
    @CacheEvict(value = "products", key = "#id")
    public void deleteProduct(Long id) {
        productRepository.remove(id);
    }

    // 모든 캐시 제거
    @CacheEvict(value = "products", allEntries = true)
    public void clearCache() {
        System.out.println("모든 캐시 삭제");
    }
}
```

 - `캐시 동작 확인`
    - 첫 번째 조회: DB에서 값을 가져오고 캐시에 저장.
    - 두 번째 조회: 캐시에서 가져오므로 DB 조회가 발생하지 않음.
    - 캐시 삭제 후 조회: 캐시에서 제거되었으므로 다시 DB 조회 발생.
```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
class ProductServiceTest {

    @Autowired
    private ProductService productService;

    @Test
    void testCache() {
        // 데이터 저장
        productService.updateProduct(1L, "Laptop");

        // 첫 번째 호출 (DB 조회 발생)
        System.out.println(productService.getProductById(1L));

        // 두 번째 호출 (캐시에서 가져옴, DB 조회 안 함)
        System.out.println(productService.getProductById(1L));

        // 캐시 삭제 후 다시 조회 (DB 조회 발생)
        productService.deleteProduct(1L);
        System.out.println(productService.getProductById(1L)); // NullPointerException 가능
    }
}
```
