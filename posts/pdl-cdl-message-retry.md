# PDL/CDL을 활용한 메시지 발행 재처리 전략

토스뱅크의 메시지 발행 재처리 아키텍처인 **PDL/CDL 패턴**을 정리하고, 아웃박스 패턴과의 차이점 및 실무 적용 시 고려사항을 다룹니다.

---

## 1. PDL/CDL 개념

### PDL (Producer Dead Letter)

**Producer 측에서 메시지 발행 실패 시 재처리하는 메커니즘**

```
[Application] --발행 실패--> [PDL Topic] --재처리--> [원본 Topic]
```

- 메시지 브로커(Kafka)로의 발행이 실패한 경우
- 네트워크 오류, 브로커 다운 등의 이유
- **PDL Topic**에 실패한 메시지를 저장 후 재발행

### CDL (Consumer Dead Letter)

**Consumer 측에서 메시지 처리 실패 시 재처리하는 메커니즘**

```
[Topic] --처리 실패--> [CDL Topic] --재처리--> [원본 Topic]
```

- 메시지는 정상적으로 전달됐으나 컨슈머의 비즈니스 로직에서 에러 발생
- DB 장애, 외부 API 호출 실패 등
- **CDL Topic**에 실패한 메시지를 보관 후 재처리

---

## 2. 토스뱅크의 PDL/CDL 적용

### 기본 구조

> **모든 토픽에 기본적으로 PDL이 적용**되어 있습니다.

```java
@Service
public class ExchangeService {
    
    private final KafkaProducer producer;
    
    @Transactional
    public void exchange(ExchangeRequest request) {
        // 1. 비즈니스 로직 수행 (DB 커밋)
        Exchange exchange = exchangeRepository.save(new Exchange(request));
        
        // 2. 이벤트 발행 (PDL 적용)
        producer.send("exchange-topic", new ExchangeEvent(exchange));
        // ↑ 발행 실패 시 자동으로 PDL Topic에 저장
    }
}
```

### PDL 메시지 재처리

- **PDL 서버**가 PDL Topic을 컨슘하며 재발행
- PDL 서버도 컨슈머로 동작하므로, 실패 시 **CDL**로 재처리 가능

```
[환전서버] --실패--> [PDL Topic]
                        ↓
                  [PDL 서버] --재발행--> [exchange-topic]
                        ↓ (재발행 실패)
                  [CDL Topic] --재처리--> [PDL Topic]
```

---

## 3. 아웃박스 패턴과의 비교

### 아웃박스 패턴

```sql
-- 트랜잭션으로 묶임
BEGIN;
  INSERT INTO exchange ...;           -- 비즈니스 데이터
  INSERT INTO outbox (event) ...;    -- 이벤트
COMMIT;
```

- 별도 배치/폴링으로 `outbox` 테이블을 읽어 메시지 발행
- **장점**: DB 커밋과 이벤트 저장이 원자적으로 처리됨
- **단점**: 모든 서비스 DB에 아웃박스 테이블 필요

### PDL 패턴

```java
@Transactional
public void exchange(...) {
    repository.save(...);
    producer.send(...);  // 커밋 후 발행
}
```

- 메시지 브로커(Kafka)에 실패 메시지 저장
- **장점**: DB 종류/물리서버 무관하게 일괄 적용 가능
- **단점**: 커밋 후 애플리케이션이 죽으면 메시지 발행 누락 가능

---

## 4. PDL의 장점 (토스뱅크 사례)

### 4.1 MSA 환경에서의 일괄 적용 용이성

토스뱅크는:
- **수백 개**의 MSA 서버
- **수십 개**의 분리된 물리 DB 서버
- **다양한 DB**: Oracle, MySQL, MongoDB 등

아웃박스 패턴 적용 시:
- DB 종류별로 아웃박스 테이블 생성
- 물리 서버별로 폴링 애플리케이션 개발
- 스키마별 관리 복잡도 증가

PDL 패턴 적용 시:
- **공통 메시지 브로커**(Kafka)만 바라보면 됨
- **라이브러리** 형태로 일괄 배포 가능
- 각 서비스는 DB 종류와 무관하게 동일한 방식으로 재처리

### 4.2 인프라 고가용성

> PDL 메시지 브로커로 **로그 Kafka 클러스터**를 사용합니다.

- 이미 모든 서버가 로그를 남기는 용도로 사용 중
- 고가용성 설정 완료 + 24시간 모니터링
- PDL 메시지 미발행 시에도 장애 즉시 감지 가능
- **추가 인프라 비용 없음**

---

## 5. PDL의 한계와 보완 전략

### 5.1 커밋 후 애플리케이션 다운 시나리오

```java
@Transactional
public void exchange(...) {
    exchangeRepository.save(...);  // ✅ DB 커밋 성공
    // ⚠️ 여기서 서버 다운 → 메시지 발행 누락
    producer.send(...);
}
```

아웃박스 패턴:
- 이미 `outbox` 테이블에 저장되어 있어 배치로 재발행 가능

PDL 패턴:
- **메시지 자체가 발행되지 않음** → PDL로 처리 불가

### 5.2 보완: 배치 기반 재발행

토스뱅크는 **추가 배치**를 구현하여 처리:

```java
@Scheduled(fixedDelay = 60000)
public void republishMissingEvents() {
    // 1. DB에서 최근 N분간 생성된 환전 건 조회
    List<Exchange> exchanges = exchangeRepository
        .findByCreatedAtAfter(now().minusMinutes(10));
    
    // 2. 메시지 브로커에서 발행 이력 확인
    Set<Long> publishedIds = getPublishedExchangeIds();
    
    // 3. 발행 안 된 건만 재발행
    exchanges.stream()
        .filter(e -> !publishedIds.contains(e.getId()))
        .forEach(e -> producer.send("exchange-topic", new ExchangeEvent(e)));
}
```

**핵심 아이디어:**
- PDL은 "발행 실패" 케이스만 처리
- 배치는 "발행 자체가 안 된" 케이스 처리
- 두 메커니즘의 조합으로 완전성 확보

---

## 6. PDL vs 아웃박스 선택 기준

| 항목 | PDL | 아웃박스 |
|------|-----|----------|
| **적용 범위** | 메시지 브로커 장애 | 전체 장애 (앱 다운 포함) |
| **일괄 적용** | 쉬움 (라이브러리) | 어려움 (DB별 테이블) |
| **인프라** | 기존 Kafka 활용 | DB별 관리 |
| **완전성** | 배치 보완 필요 | 트랜잭션으로 보장 |
| **복잡도** | 낮음 | 높음 (폴링 구현) |

### 추천 시나리오

**PDL 패턴**
- MSA 환경에서 수십~수백 개 서비스
- DB 종류가 다양 (RDBMS, NoSQL 혼용)
- 플랫폼 팀에서 중앙 집중식 관리 가능
- 메시지 발행 실패가 주요 관심사

**아웃박스 패턴**
- 소수의 핵심 서비스 (금융 거래 등)
- 트랜잭션 완전성이 최우선
- 애플리케이션 다운타임이 빈번
- DB가 단일 종류

---

## 7. 결론

PDL/CDL 패턴은 **대규모 MSA 환경에서 메시지 발행 재처리를 일괄 적용**하기에 적합한 아키텍처입니다.

핵심 포인트:

1. **PDL**: 메시지 브로커 발행 실패 시 재처리
2. **CDL**: 컨슈머 처리 실패 시 재처리
3. **아웃박스 대비 장점**: DB 종류 무관, 라이브러리로 일괄 배포 가능
4. **한계**: 커밋 후 앱 다운 시 메시지 누락 가능
5. **보완책**: 배치로 미발행 이벤트 재발행

토스뱅크처럼 **PDL + 배치**를 조합하면, 인프라 복잡도를 낮추면서도 메시지 발행 완전성을 확보할 수 있습니다.

---

## References

- [토스뱅크 메시징 시스템 발표 영상](https://www.youtube.com/watch?v=example) (13:25부터 배치 설명)
- [Outbox Pattern - Microservices.io](https://microservices.io/patterns/data/transactional-outbox.html)
- [Kafka Dead Letter Queue Pattern](https://www.confluent.io/blog/error-handling-patterns-in-kafka/)
