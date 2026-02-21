# Resilience4J Bulkhead 패턴 완전 가이드

## 목차
1. [Bulkhead란 무엇인가?](#bulkhead란-무엇인가)
2. [왜 Bulkhead가 필요한가?](#왜-bulkhead가-필요한가)
3. [Resilience4J의 두 가지 Bulkhead 구현](#resilience4j의-두-가지-bulkhead-구현)
4. [설정 방법](#설정-방법)
5. [실제 사용 예시](#실제-사용-예시)
6. [다른 패턴과의 조합](#다른-패턴과의-조합)
7. [주의사항 및 베스트 프랙티스](#주의사항-및-베스트-프랙티스)

---

## Bulkhead란 무엇인가?

### 배에서 온 개념

**Bulkhead(격벽)**는 원래 선박 설계에서 유래한 용어입니다.

```
┌─────────────────────────────────────────────────┐
│                    선박                          │
├──────────┬──────────┬──────────┬──────────┤
│  구획 1  │  구획 2  │  구획 3  │  구획 4  │
│          │   💧     │          │          │
│          │  침수!   │          │          │
└──────────┴──────────┴──────────┴──────────┘
           ↑
      격벽(Bulkhead)이 물의 확산을 막음
```

배의 바닥을 여러 구획으로 나누어 **한 구획에 물이 들어와도 다른 구획으로 퍼지지 않도록** 설계합니다. 이렇게 하면 배 전체가 침몰하는 것을 방지할 수 있습니다.

### 소프트웨어에서의 Bulkhead

소프트웨어에서 Bulkhead 패턴은 **시스템의 일부가 실패해도 전체 시스템이 다운되지 않도록** 리소스를 격리하는 방식입니다.

```
┌────────────────────────────────────────────────────────────┐
│                     애플리케이션                             │
│                                                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ Bulkhead A  │  │ Bulkhead B  │  │ Bulkhead C  │        │
│  │ (10 스레드)  │  │ (10 스레드)  │  │ (10 스레드)  │        │
│  │             │  │             │  │             │        │
│  │  결제 서비스 │  │ 사용자 서비스│  │ 알림 서비스  │        │
│  │    호출     │  │    호출     │  │    호출     │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
│        ↓                ↓                ↓                │
└────────────────────────────────────────────────────────────┘
         ↓                ↓                ↓
   ┌──────────┐    ┌──────────┐    ┌──────────┐
   │결제 서비스│    │사용자 서비스│    │알림 서비스 │
   │  (장애!)  │    │  (정상)   │    │  (정상)   │
   └──────────┘    └──────────┘    └──────────┘
```

결제 서비스에 장애가 발생해도 **Bulkhead A**만 영향을 받고, 사용자 서비스와 알림 서비스는 정상적으로 동작합니다.

---

## 왜 Bulkhead가 필요한가?

### 문제 상황: 리소스 고갈

Bulkhead 없이 운영하면 어떤 일이 발생할까요?

#### 시나리오: Redis 장애

```
┌─────────────────────────────────────────────────────────────────┐
│                    Spring Boot 애플리케이션                       │
│                                                                 │
│   Thread Pool (총 200개)                                        │
│   ┌────────────────────────────────────────────────────────┐   │
│   │ T1  T2  T3  T4  T5  ... T198  T199  T200              │   │
│   │  ↓   ↓   ↓   ↓   ↓        ↓     ↓     ↓               │   │
│   │ 모든 스레드가 Redis 응답 대기 중... (무한 대기)           │   │
│   └────────────────────────────────────────────────────────┘   │
│                                                                 │
│   새로운 요청 → 처리할 스레드 없음 → 서비스 전체 다운!           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                    ┌────────────────┐
                    │     Redis      │
                    │   (응답 없음)   │
                    └────────────────┘
```

**결과:**
- Redis 하나의 장애가 **전체 애플리케이션 다운**으로 이어짐
- Redis와 관계없는 API도 처리 불가
- 스레드 고갈로 인한 완전한 서비스 중단

### Bulkhead 적용 후

```
┌─────────────────────────────────────────────────────────────────┐
│                    Spring Boot 애플리케이션                       │
│                                                                 │
│   ┌──────────────────┐  ┌────────────────────────────────────┐ │
│   │ Redis Bulkhead   │  │      나머지 요청 처리               │ │
│   │ (최대 20 스레드)  │  │      (180개 스레드 가용)            │ │
│   │                  │  │                                    │ │
│   │ T1~T20 대기 중   │  │  정상 처리 가능!                    │ │
│   └──────────────────┘  └────────────────────────────────────┘ │
│          ↓                                                      │
│   (격리됨)                                                      │
└─────────────────────────────────────────────────────────────────┘
               ↓
     ┌────────────────┐
     │     Redis      │
     │   (응답 없음)   │
     └────────────────┘
```

**결과:**
- Redis 관련 호출만 최대 20개로 제한
- 나머지 180개 스레드는 다른 요청 처리 가능
- **시스템의 부분적 기능 유지**

---

## Resilience4J의 두 가지 Bulkhead 구현

Resilience4J는 두 가지 Bulkhead 구현을 제공합니다.

### 1. SemaphoreBulkhead (세마포어 방식)

```
┌─────────────────────────────────────────────────┐
│           SemaphoreBulkhead                     │
│                                                 │
│   세마포어 (permits = 3)                        │
│   ┌─────────────────────────────────────────┐  │
│   │  [permit] [permit] [permit]             │  │
│   │     ↓        ↓        ↓                 │  │
│   │   요청1    요청2    요청3  ← 실행 중     │  │
│   │                                         │  │
│   │   요청4, 요청5 → 대기 또는 거부          │  │
│   └─────────────────────────────────────────┘  │
│                                                 │
│   특징:                                         │
│   - 호출자의 스레드에서 직접 실행               │
│   - 낮은 오버헤드                               │
│   - 간단한 동시성 제어                          │
└─────────────────────────────────────────────────┘
```

**특징:**
- `java.util.concurrent.Semaphore` 사용
- **호출자의 스레드**에서 코드 실행
- 오버헤드가 낮음
- I/O 바운드 작업에 적합

**설정 옵션:**
| 옵션 | 기본값 | 설명 |
|------|--------|------|
| `maxConcurrentCalls` | 25 | 동시에 허용되는 최대 호출 수 |
| `maxWaitDuration` | 0 | permit을 얻기 위해 대기하는 최대 시간 |

### 2. ThreadPoolBulkhead (스레드풀 방식)

```
┌─────────────────────────────────────────────────┐
│           ThreadPoolBulkhead                    │
│                                                 │
│   ┌─────────────────────────────────────────┐  │
│   │    대기 큐 (capacity = 5)                │  │
│   │    [요청4] [요청5] [  ] [  ] [  ]       │  │
│   └──────────────────┬──────────────────────┘  │
│                      ↓                         │
│   ┌─────────────────────────────────────────┐  │
│   │    Thread Pool (size = 3)               │  │
│   │    [Thread1] [Thread2] [Thread3]        │  │
│   │       ↓          ↓          ↓           │  │
│   │     요청1      요청2      요청3         │  │
│   └─────────────────────────────────────────┘  │
│                                                 │
│   특징:                                         │
│   - 별도 스레드풀에서 실행                      │
│   - 완전한 격리 제공                            │
│   - CPU 바운드 작업에 적합                      │
└─────────────────────────────────────────────────┘
```

**특징:**
- 별도의 스레드 풀에서 코드 실행
- **완전한 격리** 제공
- 대기 큐 지원
- CPU 바운드 작업에 적합

**설정 옵션:**
| 옵션 | 기본값 | 설명 |
|------|--------|------|
| `maxThreadPoolSize` | Runtime.availableProcessors() | 최대 스레드 풀 크기 |
| `coreThreadPoolSize` | Runtime.availableProcessors() - 1 | 핵심 스레드 풀 크기 |
| `queueCapacity` | 100 | 대기 큐 용량 |
| `keepAliveDuration` | 20ms | 유휴 스레드가 종료되기 전 대기 시간 |

### 비교표

| 항목 | SemaphoreBulkhead | ThreadPoolBulkhead |
|------|-------------------|-------------------|
| **실행 스레드** | 호출자의 스레드 | 별도 스레드 풀 |
| **오버헤드** | 낮음 | 높음 (스레드 전환) |
| **격리 수준** | 논리적 격리 | 물리적 격리 |
| **대기 큐** | 없음 (maxWaitDuration으로 대기) | 있음 |
| **적합한 작업** | I/O 바운드 | CPU 바운드 |
| **리턴 타입** | 동기 | CompletionStage (비동기) |

---

## 설정 방법

### Spring Boot application.yml 설정

#### SemaphoreBulkhead 설정

```yaml
resilience4j:
  bulkhead:
    configs:
      default:
        maxConcurrentCalls: 25
        maxWaitDuration: 0
      strict:
        maxConcurrentCalls: 10
        maxWaitDuration: 500ms
    instances:
      paymentService:
        baseConfig: default
        maxConcurrentCalls: 20
      userService:
        baseConfig: strict
      notificationService:
        maxConcurrentCalls: 5
        maxWaitDuration: 100ms
```

#### ThreadPoolBulkhead 설정

```yaml
resilience4j:
  thread-pool-bulkhead:
    configs:
      default:
        maxThreadPoolSize: 10
        coreThreadPoolSize: 5
        queueCapacity: 50
        keepAliveDuration: 20ms
    instances:
      heavyProcessingService:
        baseConfig: default
        maxThreadPoolSize: 8
        coreThreadPoolSize: 4
        queueCapacity: 20
      reportService:
        maxThreadPoolSize: 4
        coreThreadPoolSize: 2
        queueCapacity: 10
```

### Java 코드 설정

#### SemaphoreBulkhead

```java
// Config 생성
BulkheadConfig config = BulkheadConfig.custom()
    .maxConcurrentCalls(20)
    .maxWaitDuration(Duration.ofMillis(500))
    .build();

// Registry에 등록
BulkheadRegistry registry = BulkheadRegistry.of(config);

// Bulkhead 인스턴스 생성
Bulkhead bulkhead = registry.bulkhead("paymentService");

// 함수 감싸기
Supplier<String> decoratedSupplier = Bulkhead
    .decorateSupplier(bulkhead, () -> paymentService.process());

// 실행
String result = decoratedSupplier.get();
```

#### ThreadPoolBulkhead

```java
// Config 생성
ThreadPoolBulkheadConfig config = ThreadPoolBulkheadConfig.custom()
    .maxThreadPoolSize(10)
    .coreThreadPoolSize(5)
    .queueCapacity(20)
    .build();

// Registry에 등록
ThreadPoolBulkheadRegistry registry = ThreadPoolBulkheadRegistry.of(config);

// Bulkhead 인스턴스 생성
ThreadPoolBulkhead bulkhead = registry.bulkhead("heavyProcessing");

// 함수 감싸기 (CompletionStage 반환)
CompletionStage<String> stage = bulkhead.executeSupplier(
    () -> heavyService.process()
);

// 결과 처리
stage.thenAccept(result -> System.out.println("Result: " + result));
```

---

## 실제 사용 예시

### 예시 1: 어노테이션 기반 사용

```java
@Service
public class PaymentService {

    private final ExternalPaymentGateway gateway;

    @Bulkhead(name = "paymentBulkhead",
              fallbackMethod = "paymentFallback")
    public PaymentResult processPayment(PaymentRequest request) {
        return gateway.process(request);
    }

    // 폴백 메서드: Bulkhead가 꽉 찼을 때 호출됨
    public PaymentResult paymentFallback(PaymentRequest request,
                                          BulkheadFullException ex) {
        log.warn("Payment bulkhead is full. Request queued for retry.");
        return PaymentResult.queued(request.getId());
    }
}
```

### 예시 2: 여러 외부 서비스 격리

```java
@Service
public class OrderService {

    // 결제 서비스 - 20개 동시 호출 제한
    @Bulkhead(name = "paymentBulkhead",
              fallbackMethod = "paymentFallback")
    public PaymentResult processPayment(Order order) {
        return paymentClient.charge(order);
    }

    // 재고 서비스 - 30개 동시 호출 제한
    @Bulkhead(name = "inventoryBulkhead",
              fallbackMethod = "inventoryFallback")
    public InventoryResult checkInventory(Order order) {
        return inventoryClient.check(order.getItems());
    }

    // 배송 서비스 - 15개 동시 호출 제한
    @Bulkhead(name = "shippingBulkhead",
              fallbackMethod = "shippingFallback")
    public ShippingResult createShipment(Order order) {
        return shippingClient.create(order);
    }
}
```

**application.yml:**
```yaml
resilience4j:
  bulkhead:
    instances:
      paymentBulkhead:
        maxConcurrentCalls: 20
        maxWaitDuration: 1s
      inventoryBulkhead:
        maxConcurrentCalls: 30
        maxWaitDuration: 500ms
      shippingBulkhead:
        maxConcurrentCalls: 15
        maxWaitDuration: 2s
```

### 예시 3: ThreadPoolBulkhead로 무거운 작업 처리

```java
@Service
public class ReportService {

    @Bulkhead(name = "reportBulkhead",
              type = Bulkhead.Type.THREADPOOL,
              fallbackMethod = "reportFallback")
    public CompletableFuture<Report> generateReport(ReportRequest request) {
        // CPU 집약적인 리포트 생성 작업
        return CompletableFuture.supplyAsync(() -> {
            return heavyReportGeneration(request);
        });
    }

    public CompletableFuture<Report> reportFallback(ReportRequest request,
                                                     Exception ex) {
        log.warn("Report generation bulkhead is full");
        return CompletableFuture.completedFuture(
            Report.pending("Too many reports in progress. Try again later.")
        );
    }
}
```

---

## 다른 패턴과의 조합

### Bulkhead + Circuit Breaker + Retry

실무에서는 여러 패턴을 조합하여 사용합니다.

```
요청 → [Retry] → [CircuitBreaker] → [Bulkhead] → 외부 서비스
                                        ↓
                                   실행 또는 거부
```

```java
@Service
public class ResilientService {

    @Retry(name = "backendService")
    @CircuitBreaker(name = "backendService",
                    fallbackMethod = "fallback")
    @Bulkhead(name = "backendService")
    public Response callBackend(Request request) {
        return backendClient.call(request);
    }

    public Response fallback(Request request, Exception ex) {
        return Response.defaultResponse();
    }
}
```

**설정:**
```yaml
resilience4j:
  retry:
    instances:
      backendService:
        maxAttempts: 3
        waitDuration: 500ms

  circuitbreaker:
    instances:
      backendService:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10s

  bulkhead:
    instances:
      backendService:
        maxConcurrentCalls: 20
        maxWaitDuration: 500ms
```

### 적용 순서

어노테이션의 적용 순서는 다음과 같습니다:
1. **Retry** (가장 바깥)
2. **CircuitBreaker**
3. **RateLimiter**
4. **Bulkhead** (가장 안쪽)

---

## 주의사항 및 베스트 프랙티스

### 1. Bulkhead 인스턴스는 싱글톤으로 관리

```java
// ❌ 잘못된 방법: 매 호출마다 새 Bulkhead 생성
public void badMethod() {
    Bulkhead bulkhead = Bulkhead.ofDefaults("myBulkhead");
    // 이러면 격리 효과가 없음!
}

// ✅ 올바른 방법: 싱글톤으로 관리
@Configuration
public class BulkheadConfig {
    @Bean
    public BulkheadRegistry bulkheadRegistry() {
        return BulkheadRegistry.ofDefaults();
    }

    @Bean
    public Bulkhead paymentBulkhead(BulkheadRegistry registry) {
        return registry.bulkhead("payment");
    }
}
```

### 2. 적절한 maxConcurrentCalls 설정

```java
// 고려해야 할 요소들:
// 1. 전체 스레드 풀 크기
// 2. 외부 서비스의 처리 능력
// 3. 요청당 평균 처리 시간

// 예: 전체 스레드 200개, 외부 서비스 3개일 때
// 각 서비스당 50-60개 할당 + 여유분 확보
resilience4j:
  bulkhead:
    instances:
      serviceA:
        maxConcurrentCalls: 50
      serviceB:
        maxConcurrentCalls: 50
      serviceC:
        maxConcurrentCalls: 50
      # 나머지 50개는 다른 작업용
```

### 3. 모니터링 설정

```java
@Component
public class BulkheadMetrics {

    public BulkheadMetrics(Bulkhead bulkhead) {
        bulkhead.getEventPublisher()
            .onCallPermitted(event ->
                log.debug("Call permitted: {}", event))
            .onCallRejected(event ->
                log.warn("Call rejected: {}", event))
            .onCallFinished(event ->
                log.debug("Call finished: {}", event));
    }
}
```

### 4. Bulkhead vs Rate Limiter 선택 가이드

| 상황 | 권장 패턴 |
|------|----------|
| 동시 요청 수 제한 (리소스 보호) | **Bulkhead** |
| 초당/분당 요청 수 제한 (API 쿼터) | **Rate Limiter** |
| 외부 API 호출량 제한 | Rate Limiter |
| 무거운 작업의 동시 실행 제한 | **Bulkhead** |
| 짧은 응답 시간 API의 남용 방지 | Rate Limiter |
| 긴 처리 시간 작업의 리소스 보호 | **Bulkhead** |

### 5. 폴백 전략

```java
// 다양한 폴백 전략

// 1. 기본값 반환
public Result fallback(Request req, BulkheadFullException ex) {
    return Result.defaultValue();
}

// 2. 캐시된 값 반환
public Result fallback(Request req, BulkheadFullException ex) {
    return cache.getLastKnownValue(req.getId());
}

// 3. 대기열에 추가
public Result fallback(Request req, BulkheadFullException ex) {
    queue.add(req);
    return Result.queued();
}

// 4. 사용자에게 재시도 요청
public Result fallback(Request req, BulkheadFullException ex) {
    throw new ServiceBusyException("Please try again later");
}
```

---

## 정리

### Bulkhead 패턴의 핵심 가치

1. **장애 격리**: 한 서비스의 문제가 전체 시스템에 영향을 미치지 않음
2. **리소스 보호**: 스레드, 커넥션 등의 리소스 고갈 방지
3. **시스템 안정성**: 부분적 기능 저하로 전체 서비스 중단 방지

### 선택 가이드

- **SemaphoreBulkhead**: 대부분의 웹 서비스 호출, I/O 바운드 작업
- **ThreadPoolBulkhead**: CPU 집약적 작업, 완전한 격리가 필요한 경우

### 기억할 점

> "Bulkhead는 배의 격벽처럼 문제를 격리합니다. 한 구역이 침수되어도 배 전체가 가라앉지 않도록!"

---

## 참고 자료

- [Spring Cloud Circuit Breaker - Bulkhead Properties Configuration](https://docs.spring.io/spring-cloud-circuitbreaker/reference/spring-cloud-circuitbreaker-resilience4j/bulkhead-properties-configuration.html)
- [Resilience4J 공식 문서 - Bulkhead](https://resilience4j.readme.io/docs/bulkhead)
- [Reflectoring - Implementing Bulkhead with Resilience4j](https://reflectoring.io/bulkhead-with-resilience4j/)
