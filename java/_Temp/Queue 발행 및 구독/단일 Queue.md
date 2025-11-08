# 단일 Queue

## 메모리 Queue 설명

 - ✅ Thread-safe: Java의 BlockingQueue를 활용한 안전한 멀티스레드 처리
 - ✅ 블로킹/논블로킹: 다양한 메시지 발행/소비 방식 지원
 - ✅ 타임아웃 지원: 지정된 시간 동안만 대기
 - ✅ 용량 제한: 메모리 오버플로우 방지
 - ✅ Spring Boot 통합: @Component로 쉽게 주입 가능
 - ✅ 타입 안전성: 제네릭을 활용한 타입 안전한 큐
```bash
com.example.queue/
├── InMemoryMessageQueue.java          # 핵심 큐 구현
├── config/
│   ├── MessageQueueConfig.java        # 큐 Bean 설정
│   ├── OrderMessage.java              # 주문 메시지 예제
│   └── EventMessage.java              # 이벤트 메시지 예제
├── service/
│   ├── MessageProducer.java           # Producer 예제
│   └── MessageConsumer.java           # Consumer 예제
└── controller/
    └── MessageQueueController.java    # REST API 테스트용
```
<br/>

 - `주의 사항`
    - 메모리 관리
        - JVM 힙 메모리에 저장되므로 큐 크기 제한 필수
        - 대용량 메시지 처리 시 OutOfMemoryError 주의
        - 애플리케이션 재시작 시 메시지 손실
    - 영속성
        - 메모리 기반이므로 메시지 영속성 없음
        - 중요한 메시지는 DB에 별도 저장 권장
        - 애플리케이션 크래시 시 복구 불가
    - 분산 환경
        - 단일 JVM 내에서만 동작
        - 멀티 인스턴스 환경에서 공유 불가
        - 분산 환경에서는 RabbitMQ, Kafka 등 사용 권장
    - 성능
        - Producer/Consumer 비율에 따라 큐 크기 조정
        - Consumer 스레드 수 적절히 설정
        - 메시지 처리 시간 모니터링
<br/>

 - `주요 API`
```java
// 발행 (Publish)
// 1. 블로킹 방식 (큐가 가득 차면 대기)
queue.publish(message);

// 2. 타임아웃 지정 (지정 시간 동안만 대기)
boolean success = queue.publish(message, 5, TimeUnit.SECONDS);

// 3. 논블로킹 방식 (큐가 가득 차면 즉시 false 반환)
boolean success = queue.tryPublish(message);


// 소비 (Consume)
// 1. 블로킹 방식 (메시지가 올 때까지 대기)
Message msg = queue.consume();

// 2. 타임아웃 지정 (지정 시간 동안만 대기)
Message msg = queue.consume(5, TimeUnit.SECONDS);

// 3. 논블로킹 방식 (메시지가 없으면 즉시 null 반환)
Message msg = queue.tryConsume();

// 유틸리티 메서드
// 큐의 첫 번째 메시지 조회 (제거하지 않음)
Message msg = queue.peek();

// 현재 큐 크기
int size = queue.size();

// 큐가 비어있는지 확인
boolean empty = queue.isEmpty();

// 남은 용량
int remaining = queue.remainingCapacity();

// 큐 비우기
queue.clear();
```
<br/>

## 메모리 Queue 구현

### 1. Queue 설정 및 구현 클래스

 - `InMemoryMessageQueue`
```java
package com.example.queue;

import org.springframework.stereotype.Component;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;

/**
 * JVM 내부 메시지 큐 구현
 * Thread-safe하며 BlockingQueue를 활용한 구현
 */
@Component
public class InMemoryMessageQueue<T> {
    
    private static final Logger log = LoggerFactory.getLogger(InMemoryMessageQueue.class);
    
    private final BlockingQueue<T> queue;
    private final int capacity;
    
    /**
     * 기본 용량 1000으로 큐 생성
     */
    public InMemoryMessageQueue() {
        this(1000);
    }
    
    /**
     * 지정된 용량으로 큐 생성
     * @param capacity 큐의 최대 용량
     */
    public InMemoryMessageQueue(int capacity) {
        this.capacity = capacity;
        this.queue = new LinkedBlockingQueue<>(capacity);
        log.info("InMemoryMessageQueue initialized with capacity: {}", capacity);
    }
    
    /**
     * 메시지를 큐에 추가 (블로킹)
     * @param message 추가할 메시지
     * @throws InterruptedException 대기 중 인터럽트 발생 시
     */
    public void publish(T message) throws InterruptedException {
        queue.put(message);
        log.debug("Message published to queue. Current size: {}", queue.size());
    }
    
    /**
     * 메시지를 큐에 추가 (타임아웃 지정)
     * @param message 추가할 메시지
     * @param timeout 타임아웃 시간
     * @param unit 시간 단위
     * @return 성공 여부
     * @throws InterruptedException 대기 중 인터럽트 발생 시
     */
    public boolean publish(T message, long timeout, TimeUnit unit) throws InterruptedException {
        boolean result = queue.offer(message, timeout, unit);
        if (result) {
            log.debug("Message published to queue. Current size: {}", queue.size());
        } else {
            log.warn("Failed to publish message - queue is full or timeout occurred");
        }
        return result;
    }
    
    /**
     * 메시지를 큐에 추가 (논블로킹, 큐가 가득 차면 false 반환)
     * @param message 추가할 메시지
     * @return 성공 여부
     */
    public boolean tryPublish(T message) {
        boolean result = queue.offer(message);
        if (result) {
            log.debug("Message published to queue. Current size: {}", queue.size());
        } else {
            log.warn("Failed to publish message - queue is full");
        }
        return result;
    }
    
    /**
     * 큐에서 메시지를 가져옴 (블로킹)
     * @return 메시지
     * @throws InterruptedException 대기 중 인터럽트 발생 시
     */
    public T consume() throws InterruptedException {
        T message = queue.take();
        log.debug("Message consumed from queue. Remaining size: {}", queue.size());
        return message;
    }
    
    /**
     * 큐에서 메시지를 가져옴 (타임아웃 지정)
     * @param timeout 타임아웃 시간
     * @param unit 시간 단위
     * @return 메시지 (타임아웃 시 null)
     * @throws InterruptedException 대기 중 인터럽트 발생 시
     */
    public T consume(long timeout, TimeUnit unit) throws InterruptedException {
        T message = queue.poll(timeout, unit);
        if (message != null) {
            log.debug("Message consumed from queue. Remaining size: {}", queue.size());
        } else {
            log.debug("No message available within timeout period");
        }
        return message;
    }
    
    /**
     * 큐에서 메시지를 가져옴 (논블로킹, 큐가 비어있으면 null 반환)
     * @return 메시지 또는 null
     */
    public T tryConsume() {
        T message = queue.poll();
        if (message != null) {
            log.debug("Message consumed from queue. Remaining size: {}", queue.size());
        }
        return message;
    }
    
    /**
     * 큐의 첫 번째 메시지를 제거하지 않고 조회
     * @return 메시지 또는 null
     */
    public T peek() {
        return queue.peek();
    }
    
    /**
     * 현재 큐의 크기
     * @return 큐에 있는 메시지 수
     */
    public int size() {
        return queue.size();
    }
    
    /**
     * 큐가 비어있는지 확인
     * @return 비어있으면 true
     */
    public boolean isEmpty() {
        return queue.isEmpty();
    }
    
    /**
     * 큐의 남은 용량
     * @return 추가 가능한 메시지 수
     */
    public int remainingCapacity() {
        return queue.remainingCapacity();
    }
    
    /**
     * 큐 비우기
     */
    public void clear() {
        int size = queue.size();
        queue.clear();
        log.info("Queue cleared. {} messages removed", size);
    }
    
    /**
     * 큐의 최대 용량
     * @return 최대 용량
     */
    public int getCapacity() {
        return capacity;
    }
}
```
<br/>

 - `MessageQueueConfig`
```java
package com.example.queue.config;

import com.example.queue.InMemoryMessageQueue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * 메시지 큐 설정
 * 타입별로 별도의 큐를 Bean으로 등록
 */
@Configuration
public class MessageQueueConfig {
    
    /**
     * String 메시지용 큐
     */
    @Bean(name = "stringMessageQueue")
    public InMemoryMessageQueue<String> stringMessageQueue() {
        return new InMemoryMessageQueue<>(1000);
    }
    
    /**
     * 주문 메시지용 큐
     */
    @Bean(name = "orderMessageQueue")
    public InMemoryMessageQueue<OrderMessage> orderMessageQueue() {
        return new InMemoryMessageQueue<>(500);
    }
    
    /**
     * 이벤트 메시지용 큐
     */
    @Bean(name = "eventMessageQueue")
    public InMemoryMessageQueue<EventMessage> eventMessageQueue() {
        return new InMemoryMessageQueue<>(2000);
    }
}
```
<br/>

 - `OrderMessage & EventMessage`
```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class OrderMessage {
    private String orderId;
    private String customerId;
    private String productName;
    private int quantity;
    private double price;
    private LocalDateTime timestamp;
    
    public OrderMessage(String orderId, String customerId, String productName, int quantity, double price) {
        this.orderId = orderId;
        this.customerId = customerId;
        this.productName = productName;
        this.quantity = quantity;
        this.price = price;
        this.timestamp = LocalDateTime.now();
    }
}

@Data
@NoArgsConstructor
@AllArgsConstructor
public class EventMessage {
    private String eventId;
    private String eventType;
    private String payload;
    private LocalDateTime timestamp;
    
    public EventMessage(String eventId, String eventType, String payload) {
        this.eventId = eventId;
        this.eventType = eventType;
        this.payload = payload;
        this.timestamp = LocalDateTime.now();
    }
}
```
<br/>

### 2. Queue 발행자

 - `MessageProducer`
```java
package com.example.queue.service;

import com.example.queue.InMemoryMessageQueue;
import com.example.queue.config.OrderMessage;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

import java.util.UUID;
import java.util.concurrent.TimeUnit;

/**
 * 메시지 Producer 예제
 */
@Service
public class MessageProducer {
    
    private static final Logger log = LoggerFactory.getLogger(MessageProducer.class);
    
    private final InMemoryMessageQueue<String> stringQueue;
    private final InMemoryMessageQueue<OrderMessage> orderQueue;
    
    public MessageProducer(
            @Qualifier("stringMessageQueue") InMemoryMessageQueue<String> stringQueue,
            @Qualifier("orderMessageQueue") InMemoryMessageQueue<OrderMessage> orderQueue) {
        this.stringQueue = stringQueue;
        this.orderQueue = orderQueue;
    }
    
    /**
     * String 메시지 발행
     */
    public void publishStringMessage(String message) {
        try {
            stringQueue.publish(message);
            log.info("Published string message: {}", message);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            log.error("Failed to publish message", e);
        }
    }
    
    /**
     * String 메시지 발행 (타임아웃 지정)
     */
    public boolean publishStringMessageWithTimeout(String message, long timeout, TimeUnit unit) {
        try {
            boolean result = stringQueue.publish(message, timeout, unit);
            if (result) {
                log.info("Published string message: {}", message);
            } else {
                log.warn("Failed to publish message due to timeout or full queue");
            }
            return result;
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            log.error("Failed to publish message", e);
            return false;
        }
    }
    
    /**
     * 주문 메시지 발행
     */
    public void publishOrder(String customerId, String productName, int quantity, double price) {
        try {
            String orderId = UUID.randomUUID().toString();
            OrderMessage order = new OrderMessage(orderId, customerId, productName, quantity, price);
            orderQueue.publish(order);
            log.info("Published order: {}", order);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            log.error("Failed to publish order", e);
        }
    }
    
    /**
     * 주문 메시지 발행 (논블로킹)
     */
    public boolean tryPublishOrder(String customerId, String productName, int quantity, double price) {
        String orderId = UUID.randomUUID().toString();
        OrderMessage order = new OrderMessage(orderId, customerId, productName, quantity, price);
        boolean result = orderQueue.tryPublish(order);
        if (result) {
            log.info("Published order: {}", order);
        } else {
            log.warn("Failed to publish order - queue is full: {}", order);
        }
        return result;
    }
}
```
<br/>

 - `MessageQueueController`
```java
package com.example.queue.controller;

import com.example.queue.InMemoryMessageQueue;
import com.example.queue.config.OrderMessage;
import com.example.queue.service.MessageProducer;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.Map;

/**
 * 메시지 큐 테스트용 REST API
 */
@RestController
@RequestMapping("/api/queue")
public class MessageQueueController {
    
    private final MessageProducer messageProducer;
    private final InMemoryMessageQueue<String> stringQueue;
    private final InMemoryMessageQueue<OrderMessage> orderQueue;
    
    public MessageQueueController(
            MessageProducer messageProducer,
            @Qualifier("stringMessageQueue") InMemoryMessageQueue<String> stringQueue,
            @Qualifier("orderMessageQueue") InMemoryMessageQueue<OrderMessage> orderQueue) {
        this.messageProducer = messageProducer;
        this.stringQueue = stringQueue;
        this.orderQueue = orderQueue;
    }
    
    /**
     * String 메시지 발행
     */
    @PostMapping("/message")
    public ResponseEntity<Map<String, Object>> publishMessage(@RequestBody Map<String, String> request) {
        String message = request.get("message");
        messageProducer.publishStringMessage(message);
        
        Map<String, Object> response = new HashMap<>();
        response.put("success", true);
        response.put("message", "Message published successfully");
        response.put("queueSize", stringQueue.size());
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * 주문 메시지 발행
     */
    @PostMapping("/order")
    public ResponseEntity<Map<String, Object>> publishOrder(@RequestBody OrderRequest request) {
        messageProducer.publishOrder(
            request.getCustomerId(),
            request.getProductName(),
            request.getQuantity(),
            request.getPrice()
        );
        
        Map<String, Object> response = new HashMap<>();
        response.put("success", true);
        response.put("message", "Order published successfully");
        response.put("queueSize", orderQueue.size());
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * 큐 상태 조회
     */
    @GetMapping("/status")
    public ResponseEntity<Map<String, Object>> getQueueStatus() {
        Map<String, Object> response = new HashMap<>();
        
        Map<String, Object> stringQueueStatus = new HashMap<>();
        stringQueueStatus.put("size", stringQueue.size());
        stringQueueStatus.put("capacity", stringQueue.getCapacity());
        stringQueueStatus.put("remainingCapacity", stringQueue.remainingCapacity());
        stringQueueStatus.put("isEmpty", stringQueue.isEmpty());
        
        Map<String, Object> orderQueueStatus = new HashMap<>();
        orderQueueStatus.put("size", orderQueue.size());
        orderQueueStatus.put("capacity", orderQueue.getCapacity());
        orderQueueStatus.put("remainingCapacity", orderQueue.remainingCapacity());
        orderQueueStatus.put("isEmpty", orderQueue.isEmpty());
        
        response.put("stringQueue", stringQueueStatus);
        response.put("orderQueue", orderQueueStatus);
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * 큐 비우기
     */
    @DeleteMapping("/clear/{queueType}")
    public ResponseEntity<Map<String, Object>> clearQueue(@PathVariable String queueType) {
        Map<String, Object> response = new HashMap<>();
        
        if ("string".equalsIgnoreCase(queueType)) {
            int size = stringQueue.size();
            stringQueue.clear();
            response.put("message", "String queue cleared");
            response.put("removedMessages", size);
        } else if ("order".equalsIgnoreCase(queueType)) {
            int size = orderQueue.size();
            orderQueue.clear();
            response.put("message", "Order queue cleared");
            response.put("removedMessages", size);
        } else {
            response.put("error", "Invalid queue type. Use 'string' or 'order'");
            return ResponseEntity.badRequest().body(response);
        }
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * 주문 요청 DTO
     */
    public static class OrderRequest {
        private String customerId;
        private String productName;
        private int quantity;
        private double price;
        
        // Getters and Setters
        public String getCustomerId() { return customerId; }
        public void setCustomerId(String customerId) { this.customerId = customerId; }
        
        public String getProductName() { return productName; }
        public void setProductName(String productName) { this.productName = productName; }
        
        public int getQuantity() { return quantity; }
        public void setQuantity(int quantity) { this.quantity = quantity; }
        
        public double getPrice() { return price; }
        public void setPrice(double price) { this.price = price; }
    }
}
```
<br/>

### 3. Queue 소비자

 - `MessageConsumer`
```java
package com.example.queue.service;

import com.example.queue.InMemoryMessageQueue;
import com.example.queue.config.OrderMessage;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

/**
 * 메시지 Consumer 예제
 * 백그라운드 스레드에서 메시지를 지속적으로 소비
 */
@Service
public class MessageConsumer {
    
    private static final Logger log = LoggerFactory.getLogger(MessageConsumer.class);
    
    private final InMemoryMessageQueue<String> stringQueue;
    private final InMemoryMessageQueue<OrderMessage> orderQueue;
    
    private ExecutorService executorService;
    private volatile boolean running = false;
    
    public MessageConsumer(
            @Qualifier("stringMessageQueue") InMemoryMessageQueue<String> stringQueue,
            @Qualifier("orderMessageQueue") InMemoryMessageQueue<OrderMessage> orderQueue) {
        this.stringQueue = stringQueue;
        this.orderQueue = orderQueue;
    }
    
    /**
     * Consumer 시작
     */
    @PostConstruct
    public void startConsumers() {
        running = true;
        executorService = Executors.newFixedThreadPool(2);
        
        // String 메시지 Consumer
        executorService.submit(this::consumeStringMessages);
        
        // Order 메시지 Consumer
        executorService.submit(this::consumeOrderMessages);
        
        log.info("Message consumers started");
    }
    
    /**
     * Consumer 중지
     */
    @PreDestroy
    public void stopConsumers() {
        running = false;
        if (executorService != null) {
            executorService.shutdown();
            try {
                if (!executorService.awaitTermination(5, TimeUnit.SECONDS)) {
                    executorService.shutdownNow();
                }
            } catch (InterruptedException e) {
                executorService.shutdownNow();
                Thread.currentThread().interrupt();
            }
        }
        log.info("Message consumers stopped");
    }
    
    /**
     * String 메시지 소비 루프
     */
    private void consumeStringMessages() {
        log.info("String message consumer thread started");
        while (running) {
            try {
                // 1초 타임아웃으로 메시지 대기
                String message = stringQueue.consume(1, TimeUnit.SECONDS);
                if (message != null) {
                    processStringMessage(message);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            } catch (Exception e) {
                log.error("Error consuming string message", e);
            }
        }
        log.info("String message consumer thread stopped");
    }
    
    /**
     * Order 메시지 소비 루프
     */
    private void consumeOrderMessages() {
        log.info("Order message consumer thread started");
        while (running) {
            try {
                // 1초 타임아웃으로 메시지 대기
                OrderMessage order = orderQueue.consume(1, TimeUnit.SECONDS);
                if (order != null) {
                    processOrder(order);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            } catch (Exception e) {
                log.error("Error consuming order message", e);
            }
        }
        log.info("Order message consumer thread stopped");
    }
    
    /**
     * String 메시지 처리
     */
    private void processStringMessage(String message) {
        log.info("Processing string message: {}", message);
        // 실제 비즈니스 로직 구현
        try {
            Thread.sleep(100); // 처리 시뮬레이션
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    /**
     * Order 메시지 처리
     */
    private void processOrder(OrderMessage order) {
        log.info("Processing order: OrderId={}, CustomerId={}, Product={}, Quantity={}, Price={}", 
                order.getOrderId(), order.getCustomerId(), order.getProductName(), 
                order.getQuantity(), order.getPrice());
        // 실제 주문 처리 로직 구현
        try {
            Thread.sleep(200); // 처리 시뮬레이션
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    /**
     * 단일 메시지 즉시 소비 (동기 방식)
     */
    public String consumeStringMessageSync() throws InterruptedException {
        return stringQueue.consume(5, TimeUnit.SECONDS);
    }
    
    /**
     * 단일 주문 즉시 소비 (동기 방식)
     */
    public OrderMessage consumeOrderSync() throws InterruptedException {
        return orderQueue.consume(5, TimeUnit.SECONDS);
    }
}
```
<br/>

### 4. 테스트 코드

 - `InMemoryMessageQueueTest`
```java
package com.example.queue;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.DisplayName;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.*;

import static org.junit.jupiter.api.Assertions.*;

/**
 * InMemoryMessageQueue 단위 테스트
 */
class InMemoryMessageQueueTest {
    
    private InMemoryMessageQueue<String> queue;
    
    @BeforeEach
    void setUp() {
        queue = new InMemoryMessageQueue<>(10);
    }
    
    @Test
    @DisplayName("메시지 발행 및 소비 테스트")
    void testPublishAndConsume() throws InterruptedException {
        // given
        String message = "Test Message";
        
        // when
        queue.publish(message);
        String consumed = queue.consume();
        
        // then
        assertEquals(message, consumed);
        assertEquals(0, queue.size());
    }
    
    @Test
    @DisplayName("여러 메시지 순차 처리 테스트")
    void testMultipleMessages() throws InterruptedException {
        // given
        List<String> messages = List.of("msg1", "msg2", "msg3");
        
        // when
        for (String msg : messages) {
            queue.publish(msg);
        }
        
        // then
        assertEquals(3, queue.size());
        for (String expected : messages) {
            assertEquals(expected, queue.consume());
        }
        assertTrue(queue.isEmpty());
    }
    
    @Test
    @DisplayName("큐 용량 초과 시 블로킹 테스트")
    void testQueueCapacity() throws InterruptedException, ExecutionException, TimeoutException {
        // given
        InMemoryMessageQueue<Integer> smallQueue = new InMemoryMessageQueue<>(2);
        
        // when
        smallQueue.publish(1);
        smallQueue.publish(2);
        
        // 큐가 가득 찬 상태에서 publish 시도 (별도 스레드에서)
        ExecutorService executor = Executors.newSingleThreadExecutor();
        Future<Boolean> future = executor.submit(() -> {
            try {
                return smallQueue.publish(3, 100, TimeUnit.MILLISECONDS);
            } catch (InterruptedException e) {
                return false;
            }
        });
        
        // then
        assertFalse(future.get(200, TimeUnit.MILLISECONDS)); // 타임아웃으로 실패해야 함
        
        // 하나 소비하고 다시 시도
        smallQueue.consume();
        assertTrue(smallQueue.publish(3, 100, TimeUnit.MILLISECONDS));
        
        executor.shutdown();
    }
    
    @Test
    @DisplayName("tryPublish 논블로킹 테스트")
    void testTryPublish() {
        // given
        InMemoryMessageQueue<String> smallQueue = new InMemoryMessageQueue<>(2);
        
        // when & then
        assertTrue(smallQueue.tryPublish("msg1"));
        assertTrue(smallQueue.tryPublish("msg2"));
        assertFalse(smallQueue.tryPublish("msg3")); // 큐가 가득 차서 실패
        
        assertEquals(2, smallQueue.size());
    }
    
    @Test
    @DisplayName("tryConsume 논블로킹 테스트")
    void testTryConsume() throws InterruptedException {
        // given
        queue.publish("message");
        
        // when & then
        assertNotNull(queue.tryConsume());
        assertNull(queue.tryConsume()); // 큐가 비어서 null 반환
    }
    
    @Test
    @DisplayName("타임아웃이 있는 consume 테스트")
    void testConsumeWithTimeout() throws InterruptedException {
        // when
        String result = queue.consume(100, TimeUnit.MILLISECONDS);
        
        // then
        assertNull(result); // 타임아웃으로 null 반환
    }
    
    @Test
    @DisplayName("peek 테스트 - 메시지 제거하지 않음")
    void testPeek() throws InterruptedException {
        // given
        String message = "Test Message";
        queue.publish(message);
        
        // when
        String peeked1 = queue.peek();
        String peeked2 = queue.peek();
        
        // then
        assertEquals(message, peeked1);
        assertEquals(message, peeked2);
        assertEquals(1, queue.size()); // 여전히 큐에 존재
    }
    
    @Test
    @DisplayName("멀티스레드 Producer-Consumer 테스트")
    void testMultiThreadedProducerConsumer() throws InterruptedException, ExecutionException {
        // given
        InMemoryMessageQueue<Integer> testQueue = new InMemoryMessageQueue<>(100);
        int messageCount = 100;
        CountDownLatch producerLatch = new CountDownLatch(1);
        CountDownLatch consumerLatch = new CountDownLatch(1);
        
        List<Integer> producedMessages = new CopyOnWriteArrayList<>();
        List<Integer> consumedMessages = new CopyOnWriteArrayList<>();
        
        // when
        // Producer 스레드
        Thread producer = new Thread(() -> {
            try {
                producerLatch.await();
                for (int i = 0; i < messageCount; i++) {
                    testQueue.publish(i);
                    producedMessages.add(i);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        // Consumer 스레드
        Thread consumer = new Thread(() -> {
            try {
                consumerLatch.await();
                for (int i = 0; i < messageCount; i++) {
                    Integer msg = testQueue.consume(1, TimeUnit.SECONDS);
                    if (msg != null) {
                        consumedMessages.add(msg);
                    }
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        producer.start();
        consumer.start();
        
        producerLatch.countDown();
        consumerLatch.countDown();
        
        producer.join(5000);
        consumer.join(5000);
        
        // then
        assertEquals(messageCount, producedMessages.size());
        assertEquals(messageCount, consumedMessages.size());
        assertEquals(producedMessages, consumedMessages);
        assertTrue(testQueue.isEmpty());
    }
    
    @Test
    @DisplayName("여러 Consumer가 동시에 메시지 처리")
    void testMultipleConsumers() throws InterruptedException {
        // given
        InMemoryMessageQueue<Integer> testQueue = new InMemoryMessageQueue<>(100);
        int messageCount = 100;
        int consumerCount = 5;
        
        // 메시지 발행
        for (int i = 0; i < messageCount; i++) {
            testQueue.publish(i);
        }
        
        // when
        ExecutorService executor = Executors.newFixedThreadPool(consumerCount);
        ConcurrentHashMap<Integer, Integer> consumedMessages = new ConcurrentHashMap<>();
        CountDownLatch latch = new CountDownLatch(messageCount);
        
        for (int i = 0; i < consumerCount; i++) {
            executor.submit(() -> {
                while (true) {
                    try {
                        Integer msg = testQueue.consume(100, TimeUnit.MILLISECONDS);
                        if (msg != null) {
                            consumedMessages.put(msg, msg);
                            latch.countDown();
                        } else {
                            break;
                        }
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            });
        }
        
        latch.await(10, TimeUnit.SECONDS);
        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.SECONDS);
        
        // then
        assertEquals(messageCount, consumedMessages.size());
        assertTrue(testQueue.isEmpty());
    }
    
    @Test
    @DisplayName("clear 테스트")
    void testClear() throws InterruptedException {
        // given
        queue.publish("msg1");
        queue.publish("msg2");
        queue.publish("msg3");
        
        // when
        queue.clear();
        
        // then
        assertTrue(queue.isEmpty());
        assertEquals(0, queue.size());
    }
    
    @Test
    @DisplayName("remainingCapacity 테스트")
    void testRemainingCapacity() throws InterruptedException {
        // given
        int capacity = 10;
        InMemoryMessageQueue<String> testQueue = new InMemoryMessageQueue<>(capacity);
        
        // when
        assertEquals(capacity, testQueue.remainingCapacity());
        
        testQueue.publish("msg1");
        testQueue.publish("msg2");
        
        // then
        assertEquals(capacity - 2, testQueue.remainingCapacity());
        assertEquals(capacity, testQueue.getCapacity());
    }
    
    @Test
    @DisplayName("Thread 안전성 테스트 - 동시 publish/consume")
    void testThreadSafety() throws InterruptedException, ExecutionException {
        // given
        InMemoryMessageQueue<Integer> testQueue = new InMemoryMessageQueue<>(1000);
        int iterations = 1000;
        ExecutorService executor = Executors.newFixedThreadPool(10);
        
        // when
        List<Future<?>> futures = new ArrayList<>();
        
        // 5개의 Producer
        for (int i = 0; i < 5; i++) {
            final int threadId = i;
            futures.add(executor.submit(() -> {
                for (int j = 0; j < iterations; j++) {
                    try {
                        testQueue.publish(threadId * iterations + j);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                }
            }));
        }
        
        // 5개의 Consumer
        ConcurrentHashMap<Integer, Boolean> consumed = new ConcurrentHashMap<>();
        for (int i = 0; i < 5; i++) {
            futures.add(executor.submit(() -> {
                for (int j = 0; j < iterations; j++) {
                    try {
                        Integer msg = testQueue.consume(1, TimeUnit.SECONDS);
                        if (msg != null) {
                            consumed.put(msg, true);
                        }
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                }
            }));
        }
        
        // 모든 작업 완료 대기
        for (Future<?> future : futures) {
            future.get();
        }
        
        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.SECONDS);
        
        // then
        assertEquals(5 * iterations, consumed.size());
        assertTrue(testQueue.isEmpty());
    }
}
```