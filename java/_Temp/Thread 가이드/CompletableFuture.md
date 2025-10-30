# CompletableFuture

CompletableFuture는 비동기 작업을 만들고, 조합하고, 예외를 처리하는 데 최적화된 Java의 Future 구현입니다. 콜백 기반 컴포지션(then*), 병렬 실행(allOf/anyOf), 타임아웃, 취소, 예외 흐름 제어까지 한 번에 다룹니다.


## 1. 관련 문서

 - Baeldung 문서: https://www.baeldung.com/java-completablefuture
 - 기초 설명 및 사용 예시: https://wbluke.tistory.com/50
 - 사용 예시: https://dev-coco.tistory.com/185
 - 사용 예시: https://11st-tech.github.io/2024/01/04/completablefuture/

<br/>

### 1-1. 치트 시트

 - `CompletableFuture 생성 및 실행`
    - supplyAsync: Supplier 인터페이스를 받아서 반환값이 존재하는 메서드
    - get / join: CompletagbleFuture가 끝나기를 기다리는 Blocking 메서드
        - get은 Checked Exception을 던지고, join은 Unchecked Exception을 던진다.
```java
private String sayMessage(String message) {
    String saidMessage = "Say " + message;
    log.info("Said Message = {}", saidMessage);
    return saidMessage;
}

@Test
void supplyAsync() {
    /* given */
    String message = "Hello";
    CompletableFuture<String> messageFuture = CompletableFuture.supplyAsync(() -> sayMessage(message));

    /* when */
    String result = messageFuture.join();

    /* then */
    assertThat(result).isEqualTo("Say Hello");
}
```
<br/>

 - `후속 작업`
    - thenApply(Function), thenCompose: 반환형이 존재하는 메서드 (변환)
    - thenAccept(Consumer), thenRun(Runnable): 반환형이 없는 메서드 (소비)
    - thenCombine, thenAcceptBoth, runAfterBoth, applyToEither: 다른 CompletableFuture를 받아서 결과 결합
```java
// 변환 처리
@Test
void thenApply() {
    /* given */
    String message = "Hello";
    CompletableFuture<String> messageFuture = CompletableFuture.supplyAsync(() -> sayMessage(message));

    /* when */
    String result = messageFuture
            .thenApply(saidMessage -> "Applied message : " + saidMessage)
            .join();

    /* then */
    assertThat(result).isEqualTo("Applied message : Say Hello");
}

// 소비 처리
@Test
void thenAccept() {
    /* given */
    String message = "Hello";
    CompletableFuture<String> messageFuture = CompletableFuture.supplyAsync(() -> sayMessage(message));

    /* when */ /* then */
    messageFuture
            .thenAccept(saidMessage -> {
                String acceptedMessage = "accepted message : " + saidMessage;
                log.info("thenAccept {}", acceptedMessage);
            })
            .join();
}
```
<br/>

 - `Thread Pool 지정`
    - 비동기 작업 실행시 Executor를 지정해주지 않으면, 후속 작업시 다른 Thread가 처리하게 된다.
    - Executor를 지정하는 경우, 후속 작업시에도 실행되었던 Thread가 다음 후속 작업을 처리하게 된다.
```java
// Thread Pool 정의
@EnableAsync
@Configuration
public class AsyncConfig {

    public static final String LEARNING_DEFAULT_EXECUTOR_NAME = "threadPoolTaskExecutor";
    private static final int POOL_SIZE = 3;

    @Bean(name = LEARNING_DEFAULT_EXECUTOR_NAME)
    public Executor threadPoolTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(POOL_SIZE);
        executor.setMaxPoolSize(POOL_SIZE);
        executor.setThreadNamePrefix("thread-");
        return executor;
    }

}

// 테스트 코드
@Slf4j
@SpringBootTest
public class CompletableFutureTest {

    @Autowired
    private Executor threadPoolTaskExecutor;

    @DisplayName("thenApply() : 처음 진행한 스레드가 쭉 이어서 진행한다.")
    @Test
    void thenApplyWithSameThread() {
        /* given */
        String message = "Hello";
        CompletableFuture<String> messageFuture = CompletableFuture.supplyAsync(
                () -> sayMessage(message), threadPoolTaskExecutor
        );

        /* when */
        String result = messageFuture
                .thenApply(saidMessage -> {
                    log.info("thenApply() : Same Thread");
                    return "Applied message : " + saidMessage;
                })
                .join();

        /* then */
        assertThat(result).isEqualTo("Applied message : Say Hello");
    }

}
```
<br/>

 - `예외 처리`
    - exceptionally: 예외 발생시 예외를 인자로 받아서 처리
    - handle: 메서드 결과와 예외 결과를 인자로 받아서 처리
        - 정상 처리시 -> Object, null
        - 예외 발생시 -> null, Object
```java
// Exceptionally
String message = "Hello";
CompletableFuture<String> messageFuture = CompletableFuture.supplyAsync(() -> sayMessage(message));

String result = messageFuture
        .handle(Throwable::getMessage)
        .join();

// Handle
String message = "Hello";
CompletableFuture<String> messageFuture = CompletableFuture.supplyAsync(() -> sayMessage(message));

String result = messageFuture
        .handle((saidMessage, error) -> {
            return error == null ? saidMessage : error.getMessage();
        })
        .join();
```
<br/>

 - `병렬 처리`
    - allOf: 여러 작업들을 동시에 실행하고, 모든 작업 결과에 콜백을 실행
    - allOf는 배열 형태의 가변인자(`CompletableFuture<?>... futures`)를 받는다.
    - allOf는 모든 Future가 끝나면 완료되는 `CompletableFuture<Void>`를 반환한다.
        - 결과값은 없고 (Void), 단순히 "모두 완료됨" 신호만 줌
```java
/*
 * 기본 사용 예시
 */ 
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> "World!");

CompletableFuture<Void> combinedFuture = CompletableFuture.allOf(future1, future2);
combinedFuture.get();

assertTrue(future1.isDone());
assertTrue(future2.isDone());

String combined = Stream.of(future1, future2)
    .map(CompletableFuture::join)
    .collect(Collectors.joining(" "));

assertEquals("Hello World!", combined);

/*
 * 응용 사용 예시
 */ 
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> "World!");
List<CompletableFuture<String>> futures = List.of(future1, future2); // future N개 -> List

// allOf에 CompletableFuture 배열을 매개변수로 설정
// futures.toArray(CompletableFuture[]::new): List -> Array
// futures.toArray(new CompletableFuture[0]): List -> Array
CompletableFuture<Void> all = CompletableFuture.allOf(futures.toArray(CompletableFuture[]::new));

List<String> result = all.thenApply(v -> futures.stream()
            .map(CompletableFuture::join) // String 추출
            .collect(Collectors.toList()) // List<String> 으로 반환
            //.collect(Collectors.joining(" ")) // String 으로 반환
        ) // thenApply -> CompletableFuture<List<String>>
        .join(); // CompletableFuture<List<String>> -> List<String>
```
<br/>

## 2. 핵심 기능 및 특징

 - __Future의 진화형__: `Future#get()`의 블로킹 한계를 극복. 콜백으로 후속 작업을 연결(`thenApply`, `thenCompose` 등)하고, 여러 작업을 조합(allOf, anyOf)할 수 있음.
 - __실행 컨텍스트__: 기본은 `ForkJoinPool.commonPool()`. `thenApplyAsync(..., executor)` 등으로 커스텀 Executor 지정 가능.
 - __상태 전이__: `Incomplete → Completed normally | Completed exceptionally | Cancelled`.
 - __함수형 단계__(3가지 계열)
    - 변환: `thenApply`, `thenCompose`
    - 소비(사이드이펙트): `thenAccept`, `thenRun`
    - 결합(두 개 이상): `thenCombine`, `thenAcceptBoth`, `runAfterBoth`, `applyToEither` 등
 - __예외 처리__: `exceptionally`, `handle`, `whenComplete`로 파이프라인에서 예외를 포착/대체/로깅.
 - __타임아웃/기한__: JDK 9+의 `orTimeout`, `completeOnTimeout` 또는 `Executor + Scheduled` 조합.
 - __취소__: `cancel(true)` → 이후 단계는 실행되지 않음(이미 실행된 단계는 예외).

<br/>

### 2-1. CompletableFuture API

#### API 요약

| 카테고리   | 메서드                                               | 설명                       |
| ------ | ------------------------------------------------- | ------------------------ |
| 시작     | `completedFuture`, `supplyAsync`, `runAsync`      | 즉시완료 / 값있는 비동기 / 값없는 비동기 |
| 변환     | `thenApply`, `thenApplyAsync`                     | 결과 변환(동기/비동기)            |
| 평탄화    | `thenCompose`, `thenComposeAsync`                 | `CF<CF<T>>` → `CF<T>`    |
| 소비     | `thenAccept`, `thenRun`                           | 값 소비/사이드이펙트              |
| 결합(2개) | `thenCombine`, `thenAcceptBoth`, `runAfterBoth`   | 두 CF 병렬 결과 결합            |
| 경쟁     | `applyToEither`, `acceptEither`, `runAfterEither` | 먼저 끝난 것 사용               |
| 다중     | `allOf`, `anyOf`                                  | 다수 병렬 조합 또는 첫 완료         |
| 예외     | `exceptionally`, `handle`, `whenComplete`         | 대체/분기/로깅/finally         |
| 타임아웃   | `orTimeout`, `completeOnTimeout`                  | 시간 초과/기본값으로 완료           |
| 제어     | `complete`, `completeExceptionally`, `cancel`     | 외부에서 상태 전이               |
| 동기 대기  | `get`, `get(timeout)`, `join`                     | 결과 대기(서비스 가장자리에서만)       |

<br/>

#### API 설명

 - `생성/시작`

| 메서드                         | 시그니처(요지)                                                                                                      | 설명 / 언제 쓰나                                                        |
| --------------------------- | ------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| `completedFuture`           | `static <T> CF<T> completedFuture(T value)`                                                                   | 이미 계산된 값을 가진 완료 상태 CF 생성. 테스트/캐시 히트 등에 유용.                        |
| `supplyAsync`               | `static <T> CF<T> supplyAsync(Supplier<T> fn)`<br>`static <T> CF<T> supplyAsync(Supplier<T> fn, Executor ex)` | __값을 반환하는__ 비동기 작업 시작. I/O나 비용 큰 계산의 진입점.                         |
| `runAsync`                  | `static CF<Void> runAsync(Runnable task)`<br>`static CF<Void> runAsync(Runnable task, Executor ex)`           | __반환값 없는__ 비동기 작업 시작. 사이드 이펙트 처리.                                 |
| `new CompletableFuture<>()` | 생성자                                                                                                           | 수동 완료용 “핸들”. 외부 이벤트/콜백에서 `complete`/`completeExceptionally`로 마무리. |

<br/>

 - `변환/연결(단계 컴포지션)`

| 메서드                                | 요지                                 | 설명 / 팁                                                                                                 |
| ---------------------------------- | ---------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `thenApply` / `thenApplyAsync`     | `CF<U> thenApply(fn T→U)`          | 이전 결과를 __동기__(기본) 또는 별도 스레드에서 __변환__. 가벼운 계산은 `thenApply`, 무거운 계산은 `thenApplyAsync(..., executor)` 권장. |
| `thenCompose` / `thenComposeAsync` | `CF<U> thenCompose(fn T→CF<U>)`    | `CF<CF<U>>` __평탄화__. __다음 단계도 비동기__일 때 필수. *(API 호출 → 그 결과로 또 API 호출)*                                 |
| `thenAccept` / `thenAcceptAsync`   | `CF<Void> thenAccept(Consumer<T>)` | 결과를 __소비(사이드 이펙트)__ 하고 다음으로. 로깅/캐시 저장 등.                                                               |
| `thenRun` / `thenRunAsync`         | `CF<Void> thenRun(Runnable)`       | 이전 결과를 쓰지 않고 __후속 작업만 실행__. 파이프라인의 “훅”.                                                                |

<br/>

 - `병렬 결합/경쟁`

| 메서드                               | 요지                                                | 설명 / 언제 쓰나                                   |
| --------------------------------- | ------------------------------------------------- | -------------------------------------------- |
| `thenCombine`                     | `CF<R> thenCombine(CF<U> other, BiFn<T,U,R>)`     | __두 작업을 병렬__로 수행하고 __두 결과를 결합__. 합치기/머지.     |
| `thenAcceptBoth`                  | `CF<Void> thenAcceptBoth(CF<U>, BiConsumer<T,U>)` | 두 결과를 받아 __소비__(반환 없음).                      |
| `runAfterBoth`                    | `CF<Void> runAfterBoth(CF<?>, Runnable)`          | 두 작업 모두 끝난 후 __결과와 무관하게__ 실행.                |
| `applyToEither`                   | `CF<U> applyToEither(CF<T> other, Fn<T,U>)`       | __먼저 끝난__ 결과 사용해 변환. “레이스에서 승자 선택”.          |
| `acceptEither` / `runAfterEither` | `CF<Void> ...Either(...)`                         | 먼저 끝난 결과를 __소비__하거나, __둘 중 하나만 끝나도 후속 실행__.  |
| `allOf`                           | `static CF<Void> allOf(CF<?>... cfs)`             | __N개 모두 완료__될 때 완료. 결과 수집은 `join()`으로 개별 회수. |
| `anyOf`                           | `static CF<Object> anyOf(CF<?>... cfs)`           | __N개 중 하나라도 완료__되면 완료. 빠른 응답/폴백에 유용.         |

<br/>

 - `예외/마무리`

| 메서드             | 요지                                            | 설명 / 차이점                                 |
| --------------- | --------------------------------------------- | ---------------------------------------- |
| `exceptionally` | `CF<T> exceptionally(Fn<Throwable,T>)`        | __예외 시 대체 값__으로 복구. 정상 경로에 영향 없음.        |
| `handle`        | `CF<U> handle(BiFn<T,Throwable,U>)`           | __성공/실패 모두 처리해 값으로 변환__. 분기+복구를 한 곳에서.   |
| `whenComplete`  | `CF<T> whenComplete(BiConsumer<T,Throwable>)` | __finally 훅__. 결과는 그대로 통과, 사이드 이펙트/로깅 용. |

<br/>

 - `타임아웃/지연 (JDK 9+)`

| 메서드                 | 요지                                                                | 설명                                                                         |
| ------------------- | ----------------------------------------------------------------- | -------------------------------------------------------------------------- |
| `orTimeout`         | `CF<T> orTimeout(long, TimeUnit)`                                 | __기한 내 미완료 시__ `TimeoutException`으로 __예외 완료__. 이어서 `exceptionally`로 폴백 처리. |
| `completeOnTimeout` | `CF<T> completeOnTimeout(T value, long, TimeUnit)`                | __기한 내 미완료 시__ 지정 __기본값으로 정상 완료__.                                         |
| `delayedExecutor`   | `static Executor delayedExecutor(long, TimeUnit, Executor base?)` | __지연 실행용 Executor__. 재시도/백오프 구현 시 유용.                                      |

 - `완료 제어/상태 조회`

| 메서드                                           | 요지                                         | 설명                                                |
| --------------------------------------------- | ------------------------------------------ | ------------------------------------------------- |
| `complete`                                    | `boolean complete(T value)`                | __외부에서 정상 완료__. 이미 완료면 `false`.                   |
| `completeExceptionally`                       | `boolean completeExceptionally(Throwable)` | __외부에서 예외 완료__.                                   |
| `cancel`                                      | `boolean cancel(boolean mayInterrupt)`     | __취소 시도__. 이후 단계는 실행 안 됨(이미 실행된 건 제외).            |
| `obtrudeValue`                                | `void obtrudeValue(T)`                     | __강제 값 주입__(상태 덮어씀). 테스트/특수 상황 외 사용 지양.           |
| `obtrudeException`                            | `void obtrudeException(Throwable)`         | __강제 예외 주입__.                                     |
| `isDone/isCompletedExceptionally/isCancelled` | 상태 검사                                      | 현재 상태 점검.                                         |
| `get()` / `get(timeout)`                      | 동기 대기                                      | 체크 예외(`ExecutionException/InterruptedException`). |
| `join()`                                      | 동기 대기                                      | 언체크 예외(`CompletionException`). 간결하지만 예외 원인 추적 주의. |
| `getNow`                                      | `T getNow(T valueIfAbsent)`                | 아직 미완료면 __기본값 즉시 반환__(대기하지 않음).                   |

<br/>

## 3. 기본 사용 예시

### 3-1. 상황별 사용 예시

 - `비동기 시작`
    - 값이 없는 작업은 runAsync(Runnable) 사용
```java
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        // 비용 큰 I/O
        sleep(200);
        return "hello";
    });
```
<br/>

 - `변환(동기/비동기)`
```java
// 같은 스레드(기본)
CompletableFuture<Integer> length = cf.thenApply(String::length);

// commonPool (또는 지정 Executor)
CompletableFuture<Integer> lengthAsync = cf.thenApplyAsync(String::length);
```
<br/>

 - `thenApply vs thenCompose`
```java
CompletableFuture<User> loadUser(long id) { ... }
CompletableFuture<List<Order>> loadOrders(User u) { ... }

// 잘못된 중첩(Future<Future<T>> 비유)
loadUser(1L).thenApply(u -> loadOrders(u));

// 평탄화(권장)
loadUser(1L).thenCompose(this::loadOrders);
```
<br/>

 - `두 결과 결합`
```java
CompletableFuture<Integer> a = getA();  // CF<Integer>
CompletableFuture<Integer> b = getB();

CompletableFuture<Integer> sum = a.thenCombine(b, Integer::sum);
```
<br/>

 - `다수 병렬 조합`
```java
List<CompletableFuture<Item>> futures = ids.stream()
    .map(this::fetchItemAsync)
    .toList();

CompletableFuture<Void> all = CompletableFuture.allOf(futures.toArray(CompletableFuture[]::new));

CompletableFuture<List<Item>> allItems = all.thenApply(v ->
    futures.stream().map(CompletableFuture::join).toList()
);
```
<br/>

 - `가장 빠른 것 사용`
```java
CompletableFuture<String> fast = cf1.applyToEither(cf2, s -> "winner: " + s);
```
<br/>

 - `예외 처리`
```java
CompletableFuture<String> safe = mightFail()
      .exceptionally(ex -> {
          log.warn("fallback {}", ex.toString());
          return "fallback";
      });

CompletableFuture<String> withHandle = mightFail()
      .handle((val, ex) -> ex == null ? val : "fallback");

CompletableFuture<String> withFinally = mightFail()
      .whenComplete((val, ex) -> log.info("done, ex={}", ex));
```
<br/>

 - `타임아웃(JDK 9+)`
```java
// 시간 초과 시 CompletionException(TimeoutException cause)
CompletableFuture<String> withTimeout = mightSlow().orTimeout(500, TimeUnit.MILLISECONDS)
               .exceptionally(ex -> "timeout-fallback");

CompletableFuture<String> completeOnTimeout = mightSlow().completeOnTimeout("default", 300, TimeUnit.MILLISECONDS);
```
<br/>

### 3-2. 실전 예제

 - `외부 API 3개 병렬 호출 → 통합 응답 생성`
```java
ExecutorService ioPool = Executors.newFixedThreadPool(20);

CompletableFuture<User> userF = CompletableFuture.supplyAsync(() -> userApi.getUser(123), ioPool);

CompletableFuture<List<Order>> ordersF = CompletableFuture.supplyAsync(() -> orderApi.getOrders(123), ioPool);

CompletableFuture<Point> pointF = CompletableFuture.supplyAsync(() -> pointApi.getPoint(123), ioPool);

CompletableFuture<Dashboard> dashboardF =
    CompletableFuture.allOf(userF, ordersF, pointF)
        .thenApply(v -> new Dashboard(
            userF.join(),            // join은 언체크 예외(CompletionException) 발생
            ordersF.join(),
            pointF.join()
        ))
        .orTimeout(800, TimeUnit.MILLISECONDS)
        .exceptionally(ex -> Dashboard.empty("partial/fallback"));

Dashboard result = dashboardF.join(); // 최종 대기(서비스 레이어 최후 구간에서만!)
```
<br/>

 - `thenCompose 체인으로 연쇄 I/O`
```java
CompletableFuture<OrderDetail> flow =
    loadUserByToken(token)
      .thenCompose(u -> loadCart(u.id()))
      .thenCompose(cart -> checkout(cart))
      .thenCompose(payment -> sendReceipt(payment))
      .exceptionally(ex -> {
          log.error("flow failed", ex);
          return OrderDetail.failed();
      });
```
<br/>

 - `캐싱 + 타임아웃 + 폴백`
```java
CompletableFuture<Product> productF =
    cache.getOrDefault(id, null) != null
      ? CompletableFuture.completedFuture(cache.get(id))
      : CompletableFuture.supplyAsync(() -> api.fetch(id), ioPool)
          .completeOnTimeout(Product.placeholder(id), 200, TimeUnit.MILLISECONDS)
          .whenComplete((p, ex) -> { if (ex == null) cache.put(id, p); });
```
<br/>

 - `스프링 MVC/웹플럭스 경계에서 사용 (블로킹 차단)`
```java
// @Service
public CompletableFuture<Profile> getProfile(long memberId, Executor ioPool) {
    return CompletableFuture.supplyAsync(() -> dao.loadProfile(memberId), ioPool)
                            .thenCompose(p -> enrichAsync(p, ioPool));
}

// @RestController
@GetMapping("/profiles/{id}")
public CompletableFuture<ResponseEntity<Profile>> get(@PathVariable long id) {
    return service.getProfile(id, ioPool)
                  .thenApply(ResponseEntity::ok)      // 컨트롤러에서 바로 CF 반환 → 서블릿 3.1 비동기
                  .exceptionally(ex -> ResponseEntity.status(500).build());
}
```
