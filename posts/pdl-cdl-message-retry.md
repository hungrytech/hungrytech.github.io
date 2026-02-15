# PDL/CDL 메시지 재처리 아키텍처 심화

분산 시스템에서 메시지 발행 및 처리의 안정성을 확보하는 것은 핵심 과제입니다. 이 글에서는 **PDL(Producer Dead Letter)**과 **CDL(Consumer Dead Letter)** 패턴의 동작 원리, 구현 전략, 그리고 아웃박스 패턴과의 차이점을 심층적으로 다룹니다.

---

## 1. Dead Letter Queue(DLQ)의 개념

### DLQ란?

**Dead Letter Queue**는 정상적으로 처리되지 못한 메시지를 보관하는 별도의 큐입니다.

```
[원본 메시지] --실패--> [Dead Letter Queue] --분석/재처리--> [원본 큐]
```

**DLQ의 목적:**
- 실패한 메시지를 손실 없이 보존
- 실패 원인 분석 및 디버깅
- 장애 복구 후 재처리
- 메인 큐의 처리 흐름을 방해하지 않음 (Poison Message 방지)

### 실패 지점에 따른 분류

메시지 실패는 **두 가지 지점**에서 발생합니다:

| 실패 지점 | 패턴 | 설명 |
|-----------|------|------|
| 발행 시점 | **PDL** | Producer가 메시지 브로커에 발행하는 데 실패 |
| 소비 시점 | **CDL** | Consumer가 메시지를 처리하는 데 실패 |

---

## 2. PDL (Producer Dead Letter)

### 2.1 PDL이란?

**Producer Dead Letter**는 메시지 발행(Produce) 시점에 실패한 메시지를 처리하는 패턴입니다.

```
[서비스 A]
     │
     │ 메시지 발행 시도
     ▼
[메시지 브로커] ←── 실패 (timeout, broker down, network error)
     │
     │ 실패 시
     ▼
[PDL Topic] ──────> [PDL 처리 서비스] ───> [원본 Topic 재발행]
```

### 2.2 PDL 발생 시나리오

**1) 네트워크 장애**
```
Producer <---X---> Kafka Broker
           네트워크 단절
```

**2) 브로커 장애**
```
Producer ---> [Kafka Broker: DOWN]
              브로커 다운, 디스크 풀, 리더 선출 중
```

**3) 타임아웃**
```
Producer ---> Broker ---> ACK 응답 지연 ---> Timeout
```

**4) 메시지 유효성 실패**
```
Producer ---> Broker ---> 메시지 크기 초과, 스키마 불일치
```

### 2.3 PDL 구현 패턴

#### 패턴 1: 동기 Fallback

```java
public class ResilientProducer {
    
    private final KafkaProducer<String, String> mainProducer;
    private final KafkaProducer<String, String> pdlProducer;
    
    public void send(String topic, String message) {
        try {
            // 원본 토픽에 발행 시도
            mainProducer.send(topic, message).get(5, TimeUnit.SECONDS);
        } catch (Exception e) {
            // 실패 시 PDL 토픽에 저장
            PDLMessage pdlMessage = PDLMessage.builder()
                .originalTopic(topic)
                .payload(message)
                .failedAt(Instant.now())
                .errorMessage(e.getMessage())
                .retryCount(0)
                .build();
            
            pdlProducer.send("pdl-topic", pdlMessage.toJson());
        }
    }
}
```

#### 패턴 2: Async Callback

```java
public void sendAsync(String topic, String message) {
    ProducerRecord<String, String> record = new ProducerRecord<>(topic, message);
    
    producer.send(record, (metadata, exception) -> {
        if (exception != null) {
            // 비동기 실패 콜백
            handleProduceFailure(topic, message, exception);
        }
    });
}

private void handleProduceFailure(String topic, String message, Exception e) {
    // 로컬 큐 또는 PDL 토픽에 저장
    pdlQueue.add(new FailedMessage(topic, message, e));
}
```

#### 패턴 3: 라이브러리 레벨 통합

```java
@Configuration
public class KafkaProducerConfig {
    
    @Bean
    public ProducerFactory<String, String> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        
        // 재시도 설정
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        config.put(ProducerConfig.RETRY_BACKOFF_MS_CONFIG, 1000);
        
        // ACK 설정 (all = 모든 ISR 복제 확인)
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        
        // 타임아웃 설정
        config.put(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, 120000);
        config.put(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG, 30000);
        
        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

### 2.4 PDL 처리 서비스 구현

```java
@Service
public class PDLProcessor {
    
    private final KafkaProducer<String, String> producer;
    private final PDLRepository pdlRepository;
    private static final int MAX_RETRY = 5;
    
    @KafkaListener(topics = "pdl-topic")
    public void processPDL(PDLMessage message) {
        try {
            // 재발행 시도
            producer.send(message.getOriginalTopic(), message.getPayload())
                    .get(10, TimeUnit.SECONDS);
            
            // 성공 시 완료 처리
            pdlRepository.markAsProcessed(message.getId());
            
        } catch (Exception e) {
            handleRetry(message, e);
        }
    }
    
    private void handleRetry(PDLMessage message, Exception e) {
        int retryCount = message.getRetryCount() + 1;
        
        if (retryCount >= MAX_RETRY) {
            // 최대 재시도 초과 → 영구 실패 처리
            pdlRepository.markAsFailed(message.getId());
            alertService.notify("PDL 영구 실패", message);
        } else {
            // 재시도 대기
            message.setRetryCount(retryCount);
            message.setNextRetryAt(calculateBackoff(retryCount));
            pdlRepository.save(message);
        }
    }
    
    private Instant calculateBackoff(int retryCount) {
        // Exponential backoff: 1s, 2s, 4s, 8s...
        long delaySeconds = (long) Math.pow(2, retryCount - 1);
        return Instant.now().plusSeconds(delaySeconds);
    }
}
```

---

## 3. CDL (Consumer Dead Letter)

### 3.1 CDL이란?

**Consumer Dead Letter**는 메시지 소비(Consume) 시점에 실패한 메시지를 처리하는 패턴입니다.

```
[대상 Topic]
     │
     │ 메시지 수신
     ▼
[컨슈머 서비스] ───> 비즈니스 로직 실패
     │
     │ 실패 시
     ▼
[CDL Topic] ──────> [CDL 처리 서비스] ───> [원본 Topic 재발행]
```

### 3.2 CDL 발생 시나리오

**1) 비즈니스 로직 예외**
```java
@KafkaListener(topics = "order-topic")
public void processOrder(OrderEvent event) {
    // DB 저장 실패, 유효성 검증 실패 등
    orderRepository.save(new Order(event)); // SQLException!
}
```

**2) 외부 서비스 호출 실패**
```java
@KafkaListener(topics = "payment-topic")
public void processPayment(PaymentEvent event) {
    // 외부 PG사 API 호출 실패
    paymentGateway.process(event); // HttpTimeoutException!
}
```

**3) 역직렬화 실패**
```java
@KafkaListener(topics = "user-topic")
public void processUser(String message) {
    // JSON 파싱 실패
    objectMapper.readValue(message, UserEvent.class); // JsonParseException!
}
```

**4) Poison Message**
```
잘못된 데이터 형식으로 인해 무한 재시도 → 컨슈머 블로킹
```

### 3.3 CDL 구현 패턴

#### 패턴 1: Spring Kafka 기본 DLT (Dead Letter Topic)

```java
@Configuration
public class KafkaConsumerConfig {
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> 
            kafkaListenerContainerFactory(
                ConsumerFactory<String, String> consumerFactory,
                KafkaTemplate<String, String> kafkaTemplate) {
        
        ConcurrentKafkaListenerContainerFactory<String, String> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        
        factory.setConsumerFactory(consumerFactory);
        
        // Dead Letter Topic 설정
        factory.setCommonErrorHandler(new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate),
            new FixedBackOff(1000L, 3L)  // 1초 간격, 3회 재시도
        ));
        
        return factory;
    }
}
```

#### 패턴 2: 커스텀 재시도 전략

```java
@Configuration
public class CustomRetryConfig {
    
    @Bean
    public DefaultErrorHandler errorHandler(KafkaTemplate<String, String> template) {
        // Exponential Backoff 설정
        ExponentialBackOffWithMaxRetries backOff = 
            new ExponentialBackOffWithMaxRetries(5);
        backOff.setInitialInterval(1000L);    // 1초
        backOff.setMultiplier(2.0);           // 2배씩 증가
        backOff.setMaxInterval(60000L);       // 최대 1분
        
        // DLT 설정
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
            template,
            (record, ex) -> new TopicPartition(
                record.topic() + ".DLT",
                record.partition()
            )
        );
        
        DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backOff);
        
        // 재시도 하지 않을 예외 지정
        handler.addNotRetryableExceptions(
            JsonParseException.class,
            DeserializationException.class
        );
        
        return handler;
    }
}
```

#### 패턴 3: 커스텀 CDL 프로세서

```java
@Service
@Slf4j
public class CDLProcessor {
    
    @KafkaListener(topicPattern = ".*\\.DLT")
    public void processCDL(
            @Payload String payload,
            @Header(KafkaHeaders.ORIGINAL_TOPIC) String originalTopic,
            @Header(KafkaHeaders.EXCEPTION_MESSAGE) String errorMessage,
            @Header(KafkaHeaders.ORIGINAL_TIMESTAMP) long timestamp) {
        
        CDLRecord record = CDLRecord.builder()
            .originalTopic(originalTopic)
            .payload(payload)
            .errorMessage(errorMessage)
            .failedAt(Instant.ofEpochMilli(timestamp))
            .build();
        
        // 실패 유형 분류 및 처리
        if (isRetryable(errorMessage)) {
            scheduleRetry(record);
        } else {
            // 영구 실패 처리 (DB 저장, 알림 등)
            cdlRepository.save(record);
            alertService.notify("CDL 처리 불가", record);
        }
    }
    
    private boolean isRetryable(String errorMessage) {
        // 일시적 오류: 재시도 가능
        return errorMessage.contains("Connection") ||
               errorMessage.contains("Timeout") ||
               errorMessage.contains("temporarily");
    }
}
```

---

## 4. PDL과 CDL의 상호 관계

### 4.1 계층적 실패 처리

PDL과 CDL은 독립적이면서도 **계층적으로 동작**합니다:

```
[서비스 A] --발행--> [Topic X] --수신--> [서비스 B]
     │                              │
     │ 실패                          │ 실패
     ▼                              ▼
 [PDL Topic]                    [CDL Topic]
     │                              │
     │ 재발행                        │ 재처리
     ▼                              ▼
 [PDL 처리기] --실패--> [CDL]  [CDL 처리기] --재발행--> [Topic X]
```

**중요**: PDL 처리기 자체도 Consumer이므로, PDL 처리 실패 시 CDL로 넘어갑니다.

### 4.2 통합 모니터링

```java
@Component
public class DeadLetterMonitor {
    
    private final MeterRegistry meterRegistry;
    
    public void recordPDL(String topic, String error) {
        meterRegistry.counter("dead_letter",
            "type", "pdl",
            "topic", topic,
            "error_type", classifyError(error)
        ).increment();
    }
    
    public void recordCDL(String topic, String error) {
        meterRegistry.counter("dead_letter",
            "type", "cdl",
            "topic", topic,
            "error_type", classifyError(error)
        ).increment();
    }
    
    // Grafana 대시보드용 쿼리:
    // sum(rate(dead_letter_total[5m])) by (type, topic, error_type)
}
```

---

## 5. 아웃박스 패턴과의 비교

### 5.1 아웃박스 패턴 개요

아웃박스 패턴은 **비즈니스 트랜잭션과 메시지 발행을 원자적으로 처리**합니다:

```java
@Transactional
public void createOrder(OrderRequest request) {
    // 1. 비즈니스 데이터 저장
    Order order = orderRepository.save(new Order(request));
    
    // 2. 아웃박스 테이블에 이벤트 저장 (같은 트랜잭션)
    OutboxEvent event = OutboxEvent.builder()
        .aggregateType("Order")
        .aggregateId(order.getId())
        .eventType("OrderCreated")
        .payload(toJson(order))
        .build();
    outboxRepository.save(event);
}
// → 트랜잭션 커밋: order + outbox_event 동시 저장
```

별도의 폴링/CDC 서비스가 `outbox` 테이블을 읽어 메시지 발행:

```java
@Scheduled(fixedDelay = 1000)
public void publishOutboxEvents() {
    List<OutboxEvent> events = outboxRepository
        .findByPublishedFalseOrderByCreatedAtAsc();
    
    for (OutboxEvent event : events) {
        try {
            kafkaTemplate.send(event.getEventType(), event.getPayload()).get();
            event.setPublished(true);
            outboxRepository.save(event);
        } catch (Exception e) {
            // 다음 주기에 재시도
        }
    }
}
```

### 5.2 패턴별 비교

| 항목 | PDL/CDL | 아웃박스 |
|------|---------|----------|
| **데이터 일관성** | 커밋 후 앱 다운 시 손실 가능 | 트랜잭션으로 보장 |
| **구현 복잡도** | 낮음 (라이브러리) | 높음 (DB 테이블 + 폴링) |
| **일괄 적용** | 쉬움 (Kafka 공통 사용) | 어려움 (DB별 설정 필요) |
| **DB 의존성** | 없음 | 있음 (outbox 테이블) |
| **실패 범위** | 메시지 브로커 장애 시 | 애플리케이션 장애 포함 모든 경우 |
| **운영 부담** | 중간 (DL Topic 모니터링) | 높음 (outbox 테이블 관리) |

### 5.3 선택 기준

**PDL/CDL이 적합한 경우:**
- 다양한 DB를 사용하는 MSA 환경
- 플랫폼 팀에서 중앙 집중식 관리
- 메시지 발행 실패가 주요 관심사
- 빠른 적용이 필요한 경우

**아웃박스가 적합한 경우:**
- 데이터 일관성이 최우선인 도메인 (금융, 결제 등)
- 애플리케이션 다운타임에도 메시지 손실 불가
- 단일 DB 종류 사용
- 트랜잭션 범위 내 엔드-투-엔드 보장 필요

---

## 6. PDL/CDL의 한계와 보완

### 6.1 핵심 한계: 커밋 후 앱 다운

```java
@Transactional
public void createOrder(OrderRequest request) {
    orderRepository.save(new Order(request));  // ✅ DB 커밋
    // ⚠️ 여기서 서버 다운 → 메시지 자체가 발행되지 않음
    kafkaTemplate.send("order-topic", orderEvent);  // ❌ 실행 안 됨
}
```

이 경우 PDL로 처리할 수 없습니다. 메시지 발행 자체가 시도되지 않았기 때문입니다.

### 6.2 보완 전략 1: Recovery 배치

```java
@Scheduled(fixedDelay = 60000)
public void recoverMissingEvents() {
    // 최근 N분간 생성된 주문 중 이벤트 미발행 건 조회
    Instant threshold = Instant.now().minus(Duration.ofMinutes(10));
    
    List<Order> orders = orderRepository
        .findByCreatedAtAfterAndEventPublishedFalse(threshold);
    
    for (Order order : orders) {
        try {
            kafkaTemplate.send("order-topic", toEvent(order)).get();
            order.setEventPublished(true);
            orderRepository.save(order);
        } catch (Exception e) {
            log.warn("Recovery 실패: {}", order.getId());
        }
    }
}
```

### 6.3 보완 전략 2: Change Data Capture (CDC)

Debezium 등의 CDC 도구를 활용:

```
[DB] --binlog--> [Debezium] --CDC 이벤트--> [Kafka]
```

- DB 변경 사항을 실시간으로 캡처
- 애플리케이션 코드 수정 불필요
- 아웃박스 패턴의 장점을 일부 흡수

### 6.4 보완 전략 3: 하이브리드 접근

핵심 도메인에는 아웃박스, 일반 도메인에는 PDL/CDL을 적용:

```
[결제 서비스] ─── 아웃박스 패턴 (데이터 일관성 중요)
[알림 서비스] ─── PDL/CDL (재시도로 충분)
[로그 서비스] ─── PDL/CDL (일부 손실 허용)
```

---

## 7. 구현 시 고려사항

### 7.1 메시지 순서 보장

DLQ 처리 시 메시지 순서가 변경될 수 있습니다:

```
원본 순서: A → B → C
B 실패 후 재처리: A → C → B (순서 변경)
```

**해결책:**
- 메시지에 시퀀스 번호 포함
- Consumer에서 순서 보정 로직 구현
- 순서가 중요한 경우 아웃박스 고려

### 7.2 멱등성 (Idempotency)

재시도로 인한 중복 처리 방지:

```java
@KafkaListener(topics = "order-topic")
public void processOrder(OrderEvent event) {
    // 멱등성 키로 중복 처리 방지
    String idempotencyKey = event.getOrderId() + "_" + event.getEventId();
    
    if (processedEventRepository.exists(idempotencyKey)) {
        log.info("이미 처리된 이벤트: {}", idempotencyKey);
        return;
    }
    
    // 비즈니스 로직 수행
    orderService.process(event);
    
    // 처리 기록
    processedEventRepository.save(idempotencyKey);
}
```

### 7.3 알림 및 에스케일레이션

```java
@Component
public class DLQAlertService {
    
    @Scheduled(fixedDelay = 60000)
    public void checkDLQBacklog() {
        long pdlCount = getTopicLag("pdl-topic");
        long cdlCount = getTopicLag("*.DLT");
        
        if (pdlCount > 100 || cdlCount > 100) {
            slackNotifier.send(String.format(
                "⚠️ DLQ 백로그 경고\nPDL: %d\nCDL: %d",
                pdlCount, cdlCount
            ));
        }
        
        if (pdlCount > 1000 || cdlCount > 1000) {
            // PagerDuty 등 온콜 알림
            pagerDuty.trigger("DLQ 임계치 초과");
        }
    }
}
```

---

## 8. 정리

### 핵심 포인트

| 패턴 | 적용 시점 | 주요 용도 |
|------|----------|----------|
| **PDL** | 메시지 발행 시 | 브로커 장애 대응 |
| **CDL** | 메시지 소비 시 | 비즈니스 로직 실패 대응 |
| **아웃박스** | 트랜잭션 커밋 시 | 완전한 일관성 보장 |

### 패턴 선택 플로우차트

```
데이터 일관성이 중요한가?
    │
    ├─ Yes → 커밋 후 앱 다운도 막아야 하나?
    │         │
    │         ├─ Yes → 아웃박스 패턴
    │         └─ No  → PDL/CDL + Recovery 배치
    │
    └─ No  → PDL/CDL
```

### 실무 적용 팁

1. **시작은 PDL/CDL로**: 빠른 적용, 낮은 복잡도
2. **Recovery 배치 필수**: PDL만으로는 커버 불가능한 케이스 존재
3. **모니터링 필수**: DLQ 백로그 임계치 알림 설정
4. **멱등성 보장**: 재시도로 인한 중복 처리 방지
5. **핵심 도메인은 아웃박스**: 금융 거래 등 손실 불가 영역

---

## References

- [Kafka Dead Letter Queue - Confluent Docs](https://docs.confluent.io/platform/current/tutorials/examples/clients/docs/kafka-commands.html)
- [Spring Kafka Error Handling](https://docs.spring.io/spring-kafka/reference/kafka/annotation-error-handling.html)
- [Transactional Outbox Pattern - microservices.io](https://microservices.io/patterns/data/transactional-outbox.html)
- [Debezium - Change Data Capture](https://debezium.io/)
