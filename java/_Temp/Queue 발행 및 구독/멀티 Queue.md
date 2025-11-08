# 멀티 Queue

## 파티셔닝 메모리 Queue 설명

 - `파티셔닝이 필요한 이유`
```
// 단일 큐
Producer → [Single Queue] → Single Consumer (순차 처리)
처리량: 초당 100개 처리 가능

// 파티셔닝 솔루션
Producer → [Partition 0] → Consumer 0
        → [Partition 1] → Consumer 1  (병렬 처리)
        → [Partition 2] → Consumer 2
        → [Partition 3] → Consumer 3
처리량: 초당 400개 처리 가능 (4배 향상)
```
<br/>

 - `순서 보장 원리`
    - 같은 키(예: customerId)를 가진 메시지는 항상 같은 파티션으로 전송
```java
// 고객 A의 주문들
publishOrder("CUST-A", "Product1", 1, 100.0);  → Partition 2
publishOrder("CUST-A", "Product2", 2, 200.0);  → Partition 2
publishOrder("CUST-A", "Product3", 1, 150.0);  → Partition 2

// 고객 B의 주문들
publishOrder("CUST-B", "Product1", 1, 100.0);  → Partition 0
publishOrder("CUST-B", "Product2", 1, 100.0);  → Partition 0

// 결과:
// - 고객 A의 주문은 Partition 2에서 순서대로 처리
// - 고객 B의 주문은 Partition 0에서 순서대로 처리
// - 두 고객의 주문은 병렬로 처리되어 전체 처리량 2배 향상
```
<br/>

 - `파일 구조`
```bash
com.example.queue/
├── PartitionedMessageQueue.java              # 파티셔닝된 큐 구현
├── config/
│   ├── PartitionedMessageQueueConfig.java    # 파티션 큐 설정
│   ├── OrderMessage.java
│   └── EventMessage.java
├── service/
│   ├── PartitionedMessageProducer.java       # 파티션 큐 Producer
│   └── PartitionedMessageConsumer.java       # 파티션별 Consumer
└── controller/
    └── PartitionedMessageQueueController.java # 테스트 API
```
<br/>

 - `주요 API`
```java
// Configuration 설정
@Configuration
public class PartitionedMessageQueueConfig {
    
    @Bean(name = "partitionedOrderQueue")
    public PartitionedMessageQueue<String, OrderMessage> partitionedOrderQueue() {
        int partitionCount = 4;  // Consumer 수와 동일하게 설정
        int queueCapacityPerPartition = 250;
        
        // 기본 해시 기반 파티션 선택
        return new PartitionedMessageQueue<>(partitionCount, queueCapacityPerPartition);
    }
}

// 메시지 발행
// 블로킹 방식
queue.publish("CUST001", orderMessage);

// 타임아웃 지정
boolean success = queue.publish("CUST001", orderMessage, 5, TimeUnit.SECONDS);

// 논블로킹 방식
boolean success = queue.tryPublish("CUST001", orderMessage);

// 메시지 소비
// 특정 파티션에서 소비 (블로킹)
OrderMessage msg = queue.consume(0);  // 파티션 0

// 타임아웃 지정
OrderMessage msg = queue.consume(0, 1, TimeUnit.SECONDS);

// 논블로킹
OrderMessage msg = queue.tryConsume(0);
```
<br/>

## 파티셔닝 메모리 Queue 구현

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

 - `PartitionedMessageQueue`
```java
package com.example.queue;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;
import java.util.function.Function;

/**
 * 파티셔닝된 메시지 큐
 * N개의 Queue를 생성하여 병렬 처리하면서 파티션 내 순서를 보장
 * 
 * @param <K> 파티션 키 타입
 * @param <V> 메시지 타입
 */
public class PartitionedMessageQueue<K, V> {
    
    private static final Logger log = LoggerFactory.getLogger(PartitionedMessageQueue.class);
    
    private final List<InMemoryMessageQueue<V>> partitions;
    private final int partitionCount;
    private final Function<K, Integer> partitionSelector;
    
    /**
     * 파티셔닝된 큐 생성
     * 
     * @param partitionCount 파티션 수 (Consumer 수와 동일하게 설정 권장)
     * @param queueCapacity 각 파티션 큐의 용량
     * @param partitionSelector 파티션 선택 함수 (키를 받아서 파티션 번호 반환)
     */
    public PartitionedMessageQueue(
            int partitionCount, 
            int queueCapacity,
            Function<K, Integer> partitionSelector) {
        
        if (partitionCount <= 0) {
            throw new IllegalArgumentException("Partition count must be positive");
        }
        
        this.partitionCount = partitionCount;
        this.partitionSelector = partitionSelector;
        this.partitions = new ArrayList<>(partitionCount);
        
        for (int i = 0; i < partitionCount; i++) {
            partitions.add(new InMemoryMessageQueue<>(queueCapacity));
        }
        
        log.info("PartitionedMessageQueue initialized with {} partitions, capacity {} each", 
                partitionCount, queueCapacity);
    }
    
    /**
     * 기본 해시 기반 파티션 선택기를 사용하는 생성자
     */
    public PartitionedMessageQueue(int partitionCount, int queueCapacity) {
        this(partitionCount, queueCapacity, key -> {
            // 기본 해시 함수 (null-safe)
            int hash = (key == null) ? 0 : key.hashCode();
            return Math.abs(hash % partitionCount);
        });
    }
    
    /**
     * 키를 기반으로 메시지를 적절한 파티션에 발행
     * 같은 키를 가진 메시지는 항상 같은 파티션으로 전송되어 순서 보장
     * 
     * @param key 파티션 키
     * @param message 메시지
     * @throws InterruptedException
     */
    public void publish(K key, V message) throws InterruptedException {
        int partition = selectPartition(key);
        partitions.get(partition).publish(message);
        log.debug("Message published to partition {} with key: {}", partition, key);
    }
    
    /**
     * 타임아웃을 지정하여 메시지 발행
     */
    public boolean publish(K key, V message, long timeout, TimeUnit unit) throws InterruptedException {
        int partition = selectPartition(key);
        boolean result = partitions.get(partition).publish(message, timeout, unit);
        if (result) {
            log.debug("Message published to partition {} with key: {}", partition, key);
        } else {
            log.warn("Failed to publish message to partition {} - timeout or full", partition);
        }
        return result;
    }
    
    /**
     * 논블로킹 방식으로 메시지 발행
     */
    public boolean tryPublish(K key, V message) {
        int partition = selectPartition(key);
        boolean result = partitions.get(partition).tryPublish(message);
        if (result) {
            log.debug("Message published to partition {} with key: {}", partition, key);
        } else {
            log.warn("Failed to publish message to partition {} - queue full", partition);
        }
        return result;
    }
    
    /**
     * 특정 파티션에서 메시지 소비 (블로킹)
     * 각 Consumer는 자신이 담당하는 파티션에서만 소비
     * 
     * @param partitionIndex 파티션 인덱스
     * @return 메시지
     * @throws InterruptedException
     */
    public V consume(int partitionIndex) throws InterruptedException {
        validatePartitionIndex(partitionIndex);
        V message = partitions.get(partitionIndex).consume();
        log.debug("Message consumed from partition {}", partitionIndex);
        return message;
    }
    
    /**
     * 특정 파티션에서 메시지 소비 (타임아웃)
     */
    public V consume(int partitionIndex, long timeout, TimeUnit unit) throws InterruptedException {
        validatePartitionIndex(partitionIndex);
        V message = partitions.get(partitionIndex).consume(timeout, unit);
        if (message != null) {
            log.debug("Message consumed from partition {}", partitionIndex);
        }
        return message;
    }
    
    /**
     * 특정 파티션에서 메시지 소비 (논블로킹)
     */
    public V tryConsume(int partitionIndex) {
        validatePartitionIndex(partitionIndex);
        V message = partitions.get(partitionIndex).tryConsume();
        if (message != null) {
            log.debug("Message consumed from partition {}", partitionIndex);
        }
        return message;
    }
    
    /**
     * 파티션 선택
     */
    private int selectPartition(K key) {
        int partition = partitionSelector.apply(key);
        // 음수나 범위를 벗어나는 경우 처리
        partition = Math.abs(partition) % partitionCount;
        return partition;
    }
    
    /**
     * 파티션 인덱스 검증
     */
    private void validatePartitionIndex(int partitionIndex) {
        if (partitionIndex < 0 || partitionIndex >= partitionCount) {
            throw new IllegalArgumentException(
                "Invalid partition index: " + partitionIndex + 
                ", must be between 0 and " + (partitionCount - 1)
            );
        }
    }
    
    /**
     * 특정 파티션의 크기
     */
    public int getPartitionSize(int partitionIndex) {
        validatePartitionIndex(partitionIndex);
        return partitions.get(partitionIndex).size();
    }
    
    /**
     * 모든 파티션의 총 메시지 수
     */
    public int getTotalSize() {
        return partitions.stream()
                .mapToInt(InMemoryMessageQueue::size)
                .sum();
    }
    
    /**
     * 특정 파티션이 비어있는지 확인
     */
    public boolean isPartitionEmpty(int partitionIndex) {
        validatePartitionIndex(partitionIndex);
        return partitions.get(partitionIndex).isEmpty();
    }
    
    /**
     * 모든 파티션이 비어있는지 확인
     */
    public boolean isEmpty() {
        return partitions.stream()
                .allMatch(InMemoryMessageQueue::isEmpty);
    }
    
    /**
     * 특정 파티션 비우기
     */
    public void clearPartition(int partitionIndex) {
        validatePartitionIndex(partitionIndex);
        partitions.get(partitionIndex).clear();
        log.info("Partition {} cleared", partitionIndex);
    }
    
    /**
     * 모든 파티션 비우기
     */
    public void clearAll() {
        partitions.forEach(InMemoryMessageQueue::clear);
        log.info("All partitions cleared");
    }
    
    /**
     * 파티션 수 조회
     */
    public int getPartitionCount() {
        return partitionCount;
    }
    
    /**
     * 각 파티션별 상태 정보
     */
    public PartitionStats getStats() {
        PartitionStats stats = new PartitionStats();
        stats.partitionCount = partitionCount;
        stats.partitionSizes = new int[partitionCount];
        stats.totalMessages = 0;
        
        for (int i = 0; i < partitionCount; i++) {
            int size = partitions.get(i).size();
            stats.partitionSizes[i] = size;
            stats.totalMessages += size;
        }
        
        return stats;
    }
    
    /**
     * 파티션 통계 정보
     */
    public static class PartitionStats {
        public int partitionCount;
        public int[] partitionSizes;
        public int totalMessages;
        
        @Override
        public String toString() {
            StringBuilder sb = new StringBuilder();
            sb.append("PartitionStats{");
            sb.append("partitionCount=").append(partitionCount);
            sb.append(", totalMessages=").append(totalMessages);
            sb.append(", sizes=[");
            for (int i = 0; i < partitionSizes.length; i++) {
                if (i > 0) sb.append(", ");
                sb.append("P").append(i).append(":").append(partitionSizes[i]);
            }
            sb.append("]}");
            return sb.toString();
        }
    }
}
```
<br/>

 - `PartitionedMessageQueueConfig`
```java
package com.example.queue.config;

import com.example.queue.InMemoryMessageQueue;
import com.example.queue.PartitionedMessageQueue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * 메시지 큐 설정
 * 단일 큐와 파티셔닝된 큐 모두 제공
 */
@Configuration
public class PartitionedMessageQueueConfig {
    
    /**
     * String 메시지용 단일 큐
     */
    @Bean(name = "stringMessageQueue")
    public InMemoryMessageQueue<String> stringMessageQueue() {
        return new InMemoryMessageQueue<>(1000);
    }
    
    /**
     * 주문 메시지용 단일 큐
     */
    @Bean(name = "orderMessageQueue")
    public InMemoryMessageQueue<OrderMessage> orderMessageQueue() {
        return new InMemoryMessageQueue<>(500);
    }
    
    /**
     * 이벤트 메시지용 단일 큐
     */
    @Bean(name = "eventMessageQueue")
    public InMemoryMessageQueue<EventMessage> eventMessageQueue() {
        return new InMemoryMessageQueue<>(2000);
    }
    
    /**
     * 파티셔닝된 주문 메시지 큐
     * 4개의 파티션으로 구성 (customerId를 키로 사용)
     * - 처리량: 단일 큐 대비 최대 4배 향상
     * - 순서 보장: 같은 customerId는 항상 같은 파티션으로 전송
     */
    @Bean(name = "partitionedOrderQueue")
    public PartitionedMessageQueue<String, OrderMessage> partitionedOrderQueue() {
        int partitionCount = 4;  // Consumer 스레드 수와 동일하게 설정
        int queueCapacityPerPartition = 250;  // 각 파티션당 용량
        
        // 기본 해시 기반 파티션 선택기 사용
        return new PartitionedMessageQueue<>(partitionCount, queueCapacityPerPartition);
    }
    
    /**
     * 파티셔닝된 이벤트 메시지 큐
     * 8개의 파티션으로 구성 (eventType을 키로 사용)
     */
    @Bean(name = "partitionedEventQueue")
    public PartitionedMessageQueue<String, EventMessage> partitionedEventQueue() {
        int partitionCount = 8;
        int queueCapacityPerPartition = 500;
        
        return new PartitionedMessageQueue<>(partitionCount, queueCapacityPerPartition);
    }
    
    /**
     * 커스텀 파티션 선택 로직을 사용하는 큐
     * 예: 특정 비즈니스 로직에 따라 파티션 선택
     */
    @Bean(name = "customPartitionedQueue")
    public PartitionedMessageQueue<String, OrderMessage> customPartitionedQueue() {
        int partitionCount = 4;
        int queueCapacityPerPartition = 250;
        
        // 커스텀 파티션 선택 로직
        // 예: 특정 고객 등급에 따라 다른 파티션 할당
        return new PartitionedMessageQueue<>(
            partitionCount, 
            queueCapacityPerPartition,
            customerId -> {
                // VIP 고객은 파티션 0으로 (우선 처리)
                if (customerId != null && customerId.startsWith("VIP")) {
                    return 0;
                }
                // 일반 고객은 해시로 분산
                return customerId == null ? 0 : Math.abs(customerId.hashCode());
            }
        );
    }
    
    /**
     * 라운드로빈 방식 파티션 선택기를 사용하는 큐
     * 키에 관계없이 순서대로 파티션에 분산
     */
    @Bean(name = "roundRobinPartitionedQueue")
    public PartitionedMessageQueue<String, String> roundRobinPartitionedQueue() {
        int partitionCount = 4;
        int queueCapacityPerPartition = 250;
        
        // 라운드로빈 카운터 (실제로는 AtomicInteger 사용 권장)
        final int[] counter = {0};
        
        return new PartitionedMessageQueue<>(
            partitionCount, 
            queueCapacityPerPartition,
            key -> {
                // 키 무시하고 라운드로빈
                synchronized (counter) {
                    return counter[0]++ % partitionCount;
                }
            }
        );
    }
}
```
<br/>

### 2. Queue 발행자

 - `PartitionedMessageProducer`
```java
package com.example.queue.service;

import com.example.queue.PartitionedMessageQueue;
import com.example.queue.config.OrderMessage;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

import java.util.UUID;
import java.util.concurrent.TimeUnit;

/**
 * 파티셔닝된 큐에 메시지를 발행하는 Producer
 * 키(customerId)를 기반으로 파티션을 선택하여 순서 보장
 */
@Service
public class PartitionedMessageProducer {
    
    private static final Logger log = LoggerFactory.getLogger(PartitionedMessageProducer.class);
    
    private final PartitionedMessageQueue<String, OrderMessage> partitionedQueue;
    
    public PartitionedMessageProducer(
            @Qualifier("partitionedOrderQueue") PartitionedMessageQueue<String, OrderMessage> partitionedQueue) {
        this.partitionedQueue = partitionedQueue;
    }
    
    /**
     * 주문 메시지 발행
     * CustomerId를 키로 사용하여 같은 고객의 주문은 같은 파티션으로 전송되어 순서 보장
     * 
     * @param customerId 고객 ID (파티션 키)
     * @param productName 상품명
     * @param quantity 수량
     * @param price 가격
     */
    public void publishOrder(String customerId, String productName, int quantity, double price) {
        try {
            String orderId = UUID.randomUUID().toString();
            OrderMessage order = new OrderMessage(orderId, customerId, productName, quantity, price);
            
            // customerId를 키로 사용하여 파티션 선택
            partitionedQueue.publish(customerId, order);
            
            log.info("Published order to partitioned queue: OrderId={}, CustomerId={}, Product={}", 
                    orderId, customerId, productName);
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            log.error("Failed to publish order for customer: {}", customerId, e);
        }
    }
    
    /**
     * 타임아웃을 지정하여 주문 발행
     */
    public boolean publishOrder(String customerId, String productName, int quantity, double price, 
                                 long timeout, TimeUnit unit) {
        try {
            String orderId = UUID.randomUUID().toString();
            OrderMessage order = new OrderMessage(orderId, customerId, productName, quantity, price);
            
            boolean result = partitionedQueue.publish(customerId, order, timeout, unit);
            
            if (result) {
                log.info("Published order to partitioned queue: OrderId={}, CustomerId={}", 
                        orderId, customerId);
            } else {
                log.warn("Failed to publish order due to timeout - CustomerId: {}", customerId);
            }
            
            return result;
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            log.error("Failed to publish order for customer: {}", customerId, e);
            return false;
        }
    }
    
    /**
     * 논블로킹 방식으로 주문 발행
     */
    public boolean tryPublishOrder(String customerId, String productName, int quantity, double price) {
        String orderId = UUID.randomUUID().toString();
        OrderMessage order = new OrderMessage(orderId, customerId, productName, quantity, price);
        
        boolean result = partitionedQueue.tryPublish(customerId, order);
        
        if (result) {
            log.info("Published order to partitioned queue: OrderId={}, CustomerId={}", 
                    orderId, customerId);
        } else {
            log.warn("Failed to publish order - queue full. CustomerId: {}", customerId);
        }
        
        return result;
    }
    
    /**
     * 특정 파티션 키로 메시지 발행 (테스트/디버깅용)
     */
    public void publishOrderWithKey(String partitionKey, String customerId, 
                                    String productName, int quantity, double price) {
        try {
            String orderId = UUID.randomUUID().toString();
            OrderMessage order = new OrderMessage(orderId, customerId, productName, quantity, price);
            
            partitionedQueue.publish(partitionKey, order);
            
            log.info("Published order with partition key '{}': OrderId={}, CustomerId={}", 
                    partitionKey, orderId, customerId);
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            log.error("Failed to publish order with partition key: {}", partitionKey, e);
        }
    }
    
    /**
     * 파티션 통계 조회
     */
    public PartitionedMessageQueue.PartitionStats getQueueStats() {
        return partitionedQueue.getStats();
    }
}
```
<br/>

 - `PartitionedMessageQueueController`
```java
package com.example.queue.controller;

import com.example.queue.PartitionedMessageQueue;
import com.example.queue.config.OrderMessage;
import com.example.queue.service.PartitionedMessageProducer;
import com.example.queue.service.PartitionedMessageConsumer;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.Map;

/**
 * 파티셔닝된 메시지 큐 테스트용 REST API
 */
@RestController
@RequestMapping("/api/partitioned-queue")
public class PartitionedMessageQueueController {
    
    private final PartitionedMessageProducer producer;
    private final PartitionedMessageConsumer consumer;
    private final PartitionedMessageQueue<String, OrderMessage> partitionedQueue;
    
    public PartitionedMessageQueueController(
            PartitionedMessageProducer producer,
            PartitionedMessageConsumer consumer,
            @Qualifier("partitionedOrderQueue") PartitionedMessageQueue<String, OrderMessage> partitionedQueue) {
        this.producer = producer;
        this.consumer = consumer;
        this.partitionedQueue = partitionedQueue;
    }
    
    /**
     * 주문 메시지 발행 (customerId가 파티션 키)
     */
    @PostMapping("/order")
    public ResponseEntity<Map<String, Object>> publishOrder(@RequestBody OrderRequest request) {
        producer.publishOrder(
            request.getCustomerId(),
            request.getProductName(),
            request.getQuantity(),
            request.getPrice()
        );
        
        Map<String, Object> response = new HashMap<>();
        response.put("success", true);
        response.put("message", "Order published to partitioned queue");
        response.put("customerId", request.getCustomerId());
        response.put("queueStats", partitionedQueue.getStats());
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * 여러 주문을 한 번에 발행 (순서 테스트용)
     */
    @PostMapping("/orders/batch")
    public ResponseEntity<Map<String, Object>> publishBatchOrders(@RequestBody BatchOrderRequest request) {
        int publishedCount = 0;
        
        for (int i = 0; i < request.getCount(); i++) {
            String productName = request.getProductName() + "-" + (i + 1);
            producer.publishOrder(
                request.getCustomerId(),
                productName,
                request.getQuantity(),
                request.getPrice()
            );
            publishedCount++;
        }
        
        Map<String, Object> response = new HashMap<>();
        response.put("success", true);
        response.put("message", "Batch orders published");
        response.put("customerId", request.getCustomerId());
        response.put("publishedCount", publishedCount);
        response.put("queueStats", partitionedQueue.getStats());
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * 파티션 통계 조회
     */
    @GetMapping("/stats")
    public ResponseEntity<Map<String, Object>> getStats() {
        Map<String, Object> response = new HashMap<>();
        
        // 큐 통계
        PartitionedMessageQueue.PartitionStats queueStats = partitionedQueue.getStats();
        Map<String, Object> queueInfo = new HashMap<>();
        queueInfo.put("partitionCount", queueStats.partitionCount);
        queueInfo.put("totalMessages", queueStats.totalMessages);
        queueInfo.put("partitionSizes", queueStats.partitionSizes);
        
        // Consumer 처리 통계
        PartitionedMessageConsumer.ProcessingStats processingStats = consumer.getStats();
        Map<String, Object> processingInfo = new HashMap<>();
        processingInfo.put("totalProcessed", processingStats.totalProcessed);
        processingInfo.put("partitionProcessedCount", processingStats.partitionProcessedCount);
        
        response.put("queue", queueInfo);
        response.put("processing", processingInfo);
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * 특정 파티션 통계 조회
     */
    @GetMapping("/stats/partition/{partitionIndex}")
    public ResponseEntity<Map<String, Object>> getPartitionStats(@PathVariable int partitionIndex) {
        if (partitionIndex < 0 || partitionIndex >= partitionedQueue.getPartitionCount()) {
            Map<String, Object> error = new HashMap<>();
            error.put("error", "Invalid partition index");
            error.put("validRange", "0 to " + (partitionedQueue.getPartitionCount() - 1));
            return ResponseEntity.badRequest().body(error);
        }
        
        Map<String, Object> response = new HashMap<>();
        response.put("partitionIndex", partitionIndex);
        response.put("queueSize", partitionedQueue.getPartitionSize(partitionIndex));
        response.put("isEmpty", partitionedQueue.isPartitionEmpty(partitionIndex));
        
        PartitionedMessageConsumer.ProcessingStats stats = consumer.getStats();
        response.put("processedCount", stats.partitionProcessedCount[partitionIndex]);
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * 특정 파티션 비우기
     */
    @DeleteMapping("/clear/partition/{partitionIndex}")
    public ResponseEntity<Map<String, Object>> clearPartition(@PathVariable int partitionIndex) {
        if (partitionIndex < 0 || partitionIndex >= partitionedQueue.getPartitionCount()) {
            Map<String, Object> error = new HashMap<>();
            error.put("error", "Invalid partition index");
            return ResponseEntity.badRequest().body(error);
        }
        
        int beforeSize = partitionedQueue.getPartitionSize(partitionIndex);
        partitionedQueue.clearPartition(partitionIndex);
        
        Map<String, Object> response = new HashMap<>();
        response.put("success", true);
        response.put("message", "Partition " + partitionIndex + " cleared");
        response.put("removedMessages", beforeSize);
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * 모든 파티션 비우기
     */
    @DeleteMapping("/clear/all")
    public ResponseEntity<Map<String, Object>> clearAll() {
        int totalSize = partitionedQueue.getTotalSize();
        partitionedQueue.clearAll();
        
        Map<String, Object> response = new HashMap<>();
        response.put("success", true);
        response.put("message", "All partitions cleared");
        response.put("removedMessages", totalSize);
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * 순서 테스트 - 같은 고객의 여러 주문을 빠르게 발행
     */
    @PostMapping("/test/order-sequence")
    public ResponseEntity<Map<String, Object>> testOrderSequence(@RequestBody SequenceTestRequest request) {
        long startTime = System.currentTimeMillis();
        
        for (int i = 0; i < request.getOrderCount(); i++) {
            producer.publishOrder(
                request.getCustomerId(),
                "Product-Sequence-" + (i + 1),
                1,
                100.0
            );
        }
        
        long endTime = System.currentTimeMillis();
        
        Map<String, Object> response = new HashMap<>();
        response.put("success", true);
        response.put("customerId", request.getCustomerId());
        response.put("ordersPublished", request.getOrderCount());
        response.put("elapsedTimeMs", endTime - startTime);
        response.put("message", "Check logs to verify order sequence in partition");
        response.put("queueStats", partitionedQueue.getStats());
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * 부하 테스트 - 여러 고객의 주문을 동시에 발행
     */
    @PostMapping("/test/load")
    public ResponseEntity<Map<String, Object>> testLoad(@RequestBody LoadTestRequest request) {
        long startTime = System.currentTimeMillis();
        int totalPublished = 0;
        
        for (int customerId = 1; customerId <= request.getCustomerCount(); customerId++) {
            String customer = "CUST" + String.format("%04d", customerId);
            
            for (int orderNum = 1; orderNum <= request.getOrdersPerCustomer(); orderNum++) {
                producer.publishOrder(
                    customer,
                    "LoadTest-Product-" + orderNum,
                    1,
                    50.0
                );
                totalPublished++;
            }
        }
        
        long endTime = System.currentTimeMillis();
        long elapsed = endTime - startTime;
        
        Map<String, Object> response = new HashMap<>();
        response.put("success", true);
        response.put("customerCount", request.getCustomerCount());
        response.put("ordersPerCustomer", request.getOrdersPerCustomer());
        response.put("totalOrdersPublished", totalPublished);
        response.put("elapsedTimeMs", elapsed);
        response.put("throughputPerSecond", elapsed > 0 ? (totalPublished * 1000.0 / elapsed) : 0);
        response.put("queueStats", partitionedQueue.getStats());
        
        return ResponseEntity.ok(response);
    }
    
    // Request DTOs
    
    public static class OrderRequest {
        private String customerId;
        private String productName;
        private int quantity;
        private double price;
        
        public String getCustomerId() { return customerId; }
        public void setCustomerId(String customerId) { this.customerId = customerId; }
        
        public String getProductName() { return productName; }
        public void setProductName(String productName) { this.productName = productName; }
        
        public int getQuantity() { return quantity; }
        public void setQuantity(int quantity) { this.quantity = quantity; }
        
        public double getPrice() { return price; }
        public void setPrice(double price) { this.price = price; }
    }
    
    public static class BatchOrderRequest {
        private String customerId;
        private String productName;
        private int quantity;
        private double price;
        private int count;
        
        public String getCustomerId() { return customerId; }
        public void setCustomerId(String customerId) { this.customerId = customerId; }
        
        public String getProductName() { return productName; }
        public void setProductName(String productName) { this.productName = productName; }
        
        public int getQuantity() { return quantity; }
        public void setQuantity(int quantity) { this.quantity = quantity; }
        
        public double getPrice() { return price; }
        public void setPrice(double price) { this.price = price; }
        
        public int getCount() { return count; }
        public void setCount(int count) { this.count = count; }
    }
    
    public static class SequenceTestRequest {
        private String customerId;
        private int orderCount;
        
        public String getCustomerId() { return customerId; }
        public void setCustomerId(String customerId) { this.customerId = customerId; }
        
        public int getOrderCount() { return orderCount; }
        public void setOrderCount(int orderCount) { this.orderCount = orderCount; }
    }
    
    public static class LoadTestRequest {
        private int customerCount;
        private int ordersPerCustomer;
        
        public int getCustomerCount() { return customerCount; }
        public void setCustomerCount(int customerCount) { this.customerCount = customerCount; }
        
        public int getOrdersPerCustomer() { return ordersPerCustomer; }
        public void setOrdersPerCustomer(int ordersPerCustomer) { this.ordersPerCustomer = ordersPerCustomer; }
    }
}
```
<br/>

### 3. Queue 소비자

 - `PartitionedMessageConsumer`
```java
package com.example.queue.service;

import com.example.queue.PartitionedMessageQueue;
import com.example.queue.config.OrderMessage;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

/**
 * 파티션별 병렬 Consumer
 * 각 Consumer는 특정 파티션만을 처리하여 순서를 보장하면서 처리량 향상
 */
@Service
public class PartitionedMessageConsumer {
    
    private static final Logger log = LoggerFactory.getLogger(PartitionedMessageConsumer.class);
    
    private final PartitionedMessageQueue<String, OrderMessage> partitionedQueue;
    
    private ExecutorService executorService;
    private volatile boolean running = false;
    private final List<ConsumerWorker> workers = new ArrayList<>();
    
    // 통계
    private final AtomicLong totalProcessed = new AtomicLong(0);
    private final long[] partitionProcessedCount;
    
    public PartitionedMessageConsumer(
            @Qualifier("partitionedOrderQueue") PartitionedMessageQueue<String, OrderMessage> partitionedQueue) {
        this.partitionedQueue = partitionedQueue;
        this.partitionProcessedCount = new long[partitionedQueue.getPartitionCount()];
    }
    
    /**
     * Consumer 시작 - 파티션 수만큼 워커 스레드 생성
     */
    @PostConstruct
    public void startConsumers() {
        running = true;
        int partitionCount = partitionedQueue.getPartitionCount();
        executorService = Executors.newFixedThreadPool(partitionCount);
        
        // 각 파티션마다 전담 Consumer 스레드 생성
        for (int i = 0; i < partitionCount; i++) {
            ConsumerWorker worker = new ConsumerWorker(i);
            workers.add(worker);
            executorService.submit(worker);
        }
        
        log.info("Started {} partition consumers", partitionCount);
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
                if (!executorService.awaitTermination(10, TimeUnit.SECONDS)) {
                    executorService.shutdownNow();
                    if (!executorService.awaitTermination(5, TimeUnit.SECONDS)) {
                        log.error("Executor did not terminate");
                    }
                }
            } catch (InterruptedException e) {
                executorService.shutdownNow();
                Thread.currentThread().interrupt();
            }
        }
        
        log.info("Stopped all partition consumers. Total processed: {}", totalProcessed.get());
        for (int i = 0; i < partitionProcessedCount.length; i++) {
            log.info("Partition {} processed: {} messages", i, partitionProcessedCount[i]);
        }
    }
    
    /**
     * 파티션별 Consumer 워커
     */
    private class ConsumerWorker implements Runnable {
        private final int partitionIndex;
        
        public ConsumerWorker(int partitionIndex) {
            this.partitionIndex = partitionIndex;
        }
        
        @Override
        public void run() {
            log.info("Consumer worker started for partition {}", partitionIndex);
            
            while (running) {
                try {
                    // 각 파티션에서 메시지 소비 (1초 타임아웃)
                    OrderMessage message = partitionedQueue.consume(partitionIndex, 1, TimeUnit.SECONDS);
                    
                    if (message != null) {
                        processOrder(message);
                        partitionProcessedCount[partitionIndex]++;
                        totalProcessed.incrementAndGet();
                    }
                    
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    log.info("Consumer worker for partition {} interrupted", partitionIndex);
                    break;
                } catch (Exception e) {
                    log.error("Error processing message in partition {}", partitionIndex, e);
                }
            }
            
            log.info("Consumer worker stopped for partition {}. Processed: {} messages", 
                    partitionIndex, partitionProcessedCount[partitionIndex]);
        }
        
        /**
         * 주문 처리 로직
         */
        private void processOrder(OrderMessage order) {
            log.info("[Partition {}] Processing order: OrderId={}, CustomerId={}, Product={}, Quantity={}", 
                    partitionIndex,
                    order.getOrderId(), 
                    order.getCustomerId(), 
                    order.getProductName(), 
                    order.getQuantity());
            
            // 실제 비즈니스 로직
            try {
                // 처리 시뮬레이션
                Thread.sleep(100);
                
                // 순서 확인을 위한 로그
                log.debug("[Partition {}] Order {} completed", partitionIndex, order.getOrderId());
                
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                log.warn("[Partition {}] Order processing interrupted: {}", partitionIndex, order.getOrderId());
            }
        }
    }
    
    /**
     * 처리 통계 조회
     */
    public ProcessingStats getStats() {
        ProcessingStats stats = new ProcessingStats();
        stats.totalProcessed = totalProcessed.get();
        stats.partitionProcessedCount = partitionProcessedCount.clone();
        stats.queueStats = partitionedQueue.getStats();
        return stats;
    }
    
    /**
     * 처리 통계 클래스
     */
    public static class ProcessingStats {
        public long totalProcessed;
        public long[] partitionProcessedCount;
        public PartitionedMessageQueue.PartitionStats queueStats;
        
        @Override
        public String toString() {
            StringBuilder sb = new StringBuilder();
            sb.append("ProcessingStats{");
            sb.append("totalProcessed=").append(totalProcessed);
            sb.append(", byPartition=[");
            for (int i = 0; i < partitionProcessedCount.length; i++) {
                if (i > 0) sb.append(", ");
                sb.append("P").append(i).append(":").append(partitionProcessedCount[i]);
            }
            sb.append("], ").append(queueStats);
            sb.append("}");
            return sb.toString();
        }
    }
}
```