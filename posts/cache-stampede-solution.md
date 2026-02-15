# Cache Stampede 문제와 해결 방법

## Cache Stampede란?

Cache Stampede(또는 Thundering Herd, Dog-pile Effect)는 캐시가 만료되는 순간 다수의 요청이 동시에 원본 데이터 소스(DB 등)에 접근하여 시스템에 과부하를 일으키는 현상입니다.

### 발생 시나리오

```
시간 T: 캐시 만료
     ↓
요청 1 → 캐시 MISS → DB 조회 시작
요청 2 → 캐시 MISS → DB 조회 시작  
요청 3 → 캐시 MISS → DB 조회 시작
  ...     (수백~수천 개의 동시 요청)
     ↓
DB 과부하 → 응답 지연 → 장애 확산
```

### 핵심 문제점

1. **동시성 폭발**: 캐시 만료 시점에 모든 요청이 DB로 향함
2. **중복 연산**: 같은 데이터를 여러 번 조회
3. **연쇄 장애**: DB 부하 → 응답 지연 → 타임아웃 → 재시도 → 더 큰 부하

---

## 해결 방법

### 1. 분산 락 (Distributed Locking)

캐시 갱신 작업에 락을 걸어 하나의 요청만 DB에 접근하도록 합니다.

```java
public Data getData(String key) {
    Data cached = cache.get(key);
    if (cached != null) {
        return cached;
    }
    
    String lockKey = "lock:" + key;
    boolean acquired = redis.set(lockKey, "1", "NX", "EX", 10);
    
    if (acquired) {
        try {
            // 락 획득 성공: DB 조회 및 캐시 갱신
            Data data = db.query(key);
            cache.set(key, data, TTL);
            return data;
        } finally {
            redis.del(lockKey);
        }
    } else {
        // 락 획득 실패: 대기 후 재시도 또는 stale 데이터 반환
        Thread.sleep(50);
        return cache.get(key); // 또는 재귀 호출
    }
}
```

**장점**: 확실한 중복 방지  
**단점**: 락 경합, 락 획득 실패 시 처리 필요

---

### 2. XFetch / Probabilistic Early Expiration (PER)

캐시 만료 전에 확률적으로 미리 갱신합니다. TTL이 가까워질수록 갱신 확률이 높아집니다.

```python
def xfetch(key, ttl, beta=1.0):
    value, delta, expiry = cache.get_with_metadata(key)
    
    # 확률적 조기 갱신 판단
    # expiry - now: 남은 TTL
    # delta: 마지막 계산 소요 시간
    # beta: 조정 파라미터 (클수록 더 일찍 갱신)
    
    gap = expiry - current_time()
    should_recompute = gap < delta * beta * log(random())
    
    if should_recompute or value is None:
        start = current_time()
        value = compute_value(key)  # DB 조회
        delta = current_time() - start
        cache.set(key, value, delta, ttl)
    
    return value
```

**핵심 공식**:
```
gap < δ × β × ln(random())

gap: 만료까지 남은 시간
δ (delta): 계산 소요 시간
β (beta): 조기 갱신 강도 (1.0 권장)
```

**장점**: 락 없이 자연스러운 분산, 무중단 갱신  
**단점**: 확률적이라 완벽한 보장 없음

---

### 3. Singleflight 패턴

동일 키에 대한 동시 요청을 하나로 합쳐 처리합니다.

```go
import "golang.org/x/sync/singleflight"

var group singleflight.Group

func GetData(key string) (interface{}, error) {
    // 같은 key로 동시에 호출되면 하나만 실행되고 결과 공유
    result, err, _ := group.Do(key, func() (interface{}, error) {
        return db.Query(key)
    })
    return result, err
}
```

**Java 구현 (CompletableFuture 활용)**:
```java
private final ConcurrentMap<String, CompletableFuture<Data>> inFlight = 
    new ConcurrentHashMap<>();

public Data getData(String key) {
    CompletableFuture<Data> future = inFlight.computeIfAbsent(key, k -> {
        CompletableFuture<Data> f = CompletableFuture.supplyAsync(() -> {
            try {
                return db.query(k);
            } finally {
                inFlight.remove(k);
            }
        });
        return f;
    });
    return future.join();
}
```

**장점**: 구현 간단, 효과적인 중복 제거  
**단점**: 단일 노드 내에서만 동작 (분산 환경에서는 Redis 락과 결합 필요)

---

### 4. Stale-While-Revalidate (SWR)

만료된 캐시를 즉시 반환하고, 백그라운드에서 갱신합니다.

```java
public Data getData(String key) {
    CacheEntry entry = cache.getEntry(key);
    
    if (entry == null) {
        // 캐시 없음: 동기적으로 조회
        return fetchAndCache(key);
    }
    
    if (entry.isExpired()) {
        // 만료됨: stale 데이터 반환 + 비동기 갱신
        asyncRefresh(key);
        return entry.getValue();  // stale but fast
    }
    
    return entry.getValue();
}

private void asyncRefresh(String key) {
    executor.submit(() -> {
        if (refreshLock.tryLock(key)) {
            try {
                fetchAndCache(key);
            } finally {
                refreshLock.unlock(key);
            }
        }
    });
}
```

**장점**: 사용자 응답 지연 없음  
**단점**: 일시적으로 오래된 데이터 반환

---

### 5. TTL Jitter (만료 시간 분산)

캐시 TTL에 랜덤 값을 추가하여 만료 시점을 분산시킵니다.

```java
public void cacheWithJitter(String key, Data data, int baseTtl) {
    // 기본 TTL의 ±10% 범위에서 랜덤
    int jitter = (int) (baseTtl * 0.1 * (Math.random() * 2 - 1));
    int actualTtl = baseTtl + jitter;
    
    cache.set(key, data, actualTtl);
}

// 예: baseTtl = 300초
// actualTtl = 270초 ~ 330초 사이 랜덤
```

**장점**: 구현 매우 간단  
**단점**: 단독으로는 완전한 해결책이 아님

---

### 6. Proactive Cache Warming (선제적 캐시 워밍)

스케줄러가 캐시 만료 전에 미리 갱신합니다.

```java
@Scheduled(fixedRate = 60000)  // 1분마다 실행
public void warmCache() {
    List<String> hotKeys = getHotKeys();  // 인기 키 목록
    
    for (String key : hotKeys) {
        Long ttl = cache.ttl(key);
        
        // TTL이 임계값(60초) 이하면 미리 갱신
        if (ttl != null && ttl < 60) {
            Data freshData = db.query(key);
            cache.set(key, freshData, BASE_TTL);
            log.info("Proactively warmed cache for key: {}", key);
        }
    }
}
```

**장점**: 요청 트래픽과 무관하게 안정적  
**단점**: 핫 키 관리 필요, 불필요한 갱신 가능성

---

## 솔루션 비교

| 방법 | 복잡도 | 일관성 | 응답 속도 | 분산 환경 |
|------|--------|--------|-----------|----------|
| 분산 락 | 중 | 높음 | 대기 발생 | O |
| XFetch/PER | 중 | 중간 | 빠름 | O |
| Singleflight | 낮음 | 높음 | 대기 발생 | △ |
| SWR | 중 | 낮음 | 매우 빠름 | O |
| TTL Jitter | 낮음 | - | - | O |
| Proactive Warming | 중 | 높음 | 빠름 | O |

---

## SWR의 Stale 데이터 문제와 해결

SWR + TTL Jitter + Proactive Warming 조합은 응답 속도 면에서 우수하지만, **Stale 데이터를 반환하는 것이 본질적인 Trade-off**입니다. 잘못된 데이터를 보여줄 수 있다는 위험이 존재합니다.

### Stale이 문제가 되는 데이터

| 데이터 유형 | 예시 | Stale 위험도 |
|-------------|------|-------------|
| 금융 데이터 | 계좌 잔액, 주문 금액 | 치명적 |
| 재고 데이터 | 상품 수량, 좌석 수 | 높음 |
| 상태 데이터 | 온라인/오프라인, 주문 상태 | 높음 |
| 권한 데이터 | 접근 권한, 토큰 유효성 | 높음 |
| 콘텐츠 데이터 | 상품 설명, 프로필 | 낮음 |
| 정적 데이터 | 카테고리, 설정값 | 매우 낮음 |

### 해결 방법 1: Stale 허용 시간 제한 (stale-max-age)

무한정 stale을 허용하지 않고, 최대 허용 시간을 설정합니다.

```java
public Data getData(String key) {
    CacheEntry entry = cache.getEntry(key);
    
    if (entry == null) {
        return fetchAndCache(key);
    }
    
    long staleTime = System.currentTimeMillis() - entry.getExpiredAt();
    
    if (staleTime > STALE_MAX_AGE) {
        // stale 허용 시간 초과: 동기적으로 새 데이터 조회 (차단)
        return fetchAndCache(key);
    }
    
    if (entry.isExpired()) {
        // stale 허용 범위 내: 비동기 갱신 + stale 반환
        asyncRefresh(key);
        return entry.getValue();
    }
    
    return entry.getValue();
}
```

**타임라인 시각화**:
```
|-------- TTL --------|-- stale 허용 --|-- 차단 --|
                    만료            stale-max-age
                     ↓                    ↓
                 SWR 동작            동기 조회 강제
```

### 해결 방법 2: 데이터 중요도별 전략 분리

모든 데이터에 SWR을 적용하지 않습니다. 데이터 특성에 따라 전략을 다르게 적용합니다.

```java
public enum CacheStrategy {
    STRICT,      // 절대 stale 불가 (잔액, 재고)
    SWR,         // stale 허용 (상품 정보, 프로필)
    AGGRESSIVE   // 긴 stale 허용 (정적 콘텐츠)
}

public Data getData(String key, CacheStrategy strategy) {
    return switch (strategy) {
        case STRICT -> getStrict(key);      // 항상 최신 보장
        case SWR -> getSWR(key);            // stale 허용
        case AGGRESSIVE -> getAggressive(key);
    };
}

// STRICT: 캐시 없거나 만료 시 무조건 DB 조회
private Data getStrict(String key) {
    Data cached = cache.get(key);
    if (cached == null || isExpired(key)) {
        return fetchAndCache(key);
    }
    return cached;
}
```

**전략별 적용 예시**:
| 전략 | 데이터 예시 | 특징 |
|------|------------|------|
| STRICT | 계좌 잔액, 재고, 권한 | SWR 사용 안함, 분산 락 적용 |
| SWR | 상품 상세, 사용자 프로필 | stale 허용, 빠른 응답 |
| AGGRESSIVE | 카테고리 목록, 설정값 | 긴 stale 허용, 최소 갱신 |

### 해결 방법 3: Write-Through Invalidation (이벤트 기반 무효화)

데이터 변경 시 즉시 캐시를 무효화하여 stale 기간을 최소화합니다.

```java
@Transactional
public void updateProduct(Product product) {
    db.update(product);
    
    // 방법 1: 즉시 삭제 (다음 조회 시 새로 로드)
    cache.delete("product:" + product.getId());
    
    // 방법 2: 즉시 갱신 (Write-through)
    cache.set("product:" + product.getId(), product, TTL);
    
    // 방법 3: 이벤트 발행 (분산 환경)
    eventPublisher.publish(new ProductUpdatedEvent(product.getId()));
}

// 이벤트 구독 (다른 서버들도 캐시 무효화)
@EventListener
public void onProductUpdated(ProductUpdatedEvent event) {
    cache.delete("product:" + event.getProductId());
}
```

**핵심 인사이트**: 읽기에서 stale을 허용하되, **쓰기 시점에 즉시 무효화**하면 실제 stale 기간이 대폭 줄어듭니다.

### 해결 방법 4: Proactive Warming 주기 최적화

Stale 발생 자체를 줄이기 위해 Warming 주기를 조정합니다.

```java
@Scheduled(fixedRate = 30000)  // 30초마다 (더 빈번하게)
public void warmCache() {
    for (String key : getHotKeys()) {
        Long ttl = cache.ttl(key);
        
        // TTL이 전체의 20% 이하로 남으면 미리 갱신
        // 예: 300초 TTL → 60초 남았을 때 갱신
        if (ttl != null && ttl < BASE_TTL * 0.2) {
            refreshAsync(key);
        }
    }
}
```

**Warming 주기와 Stale 확률 관계**:
```
TTL: 300초, Warming 임계값: 60초

Warming 주기 30초 → Stale 확률 낮음 (거의 항상 갱신됨)
Warming 주기 60초 → Stale 확률 중간
Warming 주기 120초 → Stale 확률 높음 (놓칠 수 있음)
```

### 해결 방법 5: 버전/타임스탬프 제공 (클라이언트 판단)

Stale 데이터임을 명시하고 클라이언트가 판단하게 합니다.

```java
public class CacheResponse<T> {
    private T data;
    private long cachedAt;      // 캐시된 시점
    private boolean isStale;    // stale 여부
    private long staleSeconds;  // stale 경과 시간
}

// 클라이언트/프론트엔드에서 판단
if (response.isStale() && response.getStaleSeconds() > 30) {
    showWarning("데이터가 최신이 아닐 수 있습니다");
    // 또는 강제 새로고침 트리거
}
```

---

## Trade-off 분석: 일관성 vs 가용성

Cache Stampede 해결책을 선택할 때 핵심은 **CAP 이론의 일관성(Consistency)과 가용성(Availability) 사이의 Trade-off**입니다.

### Trade-off 스펙트럼

```
강한 일관성                                      높은 가용성
    ←─────────────────────────────────────────────→
    │                                             │
 분산 락              XFetch           SWR + Warming
 Singleflight                        TTL Jitter
    │                                             │
 최신 데이터 보장        균형점          빠른 응답 보장
 응답 지연 발생                        stale 허용
```

### 상황별 Trade-off 판단 기준

| 판단 기준 | 일관성 우선 | 가용성 우선 |
|-----------|------------|------------|
| 데이터 변경 빈도 | 자주 변경됨 | 거의 변경 없음 |
| 잘못된 데이터 비용 | 비용 큼 (금융, 재고) | 비용 낮음 (프로필) |
| 사용자 기대 | 정확한 정보 필요 | 빠른 응답 선호 |
| 트래픽 패턴 | 낮은 동시성 | 높은 동시성 |
| 복구 가능성 | 복구 어려움 | 재시도/새로고침 가능 |

### 실전 적용 매트릭스

```
┌─────────────────────────────────────────────────────────┐
│                  데이터 중요도별 전략                     │
├─────────────────┬───────────────────────────────────────┤
│   CRITICAL      │  분산 락 + Singleflight               │
│   (잔액, 재고)   │  → SWR 절대 사용 안함                  │
│                 │  → Write-Through 필수                 │
├─────────────────┼───────────────────────────────────────┤
│   IMPORTANT     │  SWR + stale-max-age (짧게)           │
│   (주문 상태)    │  + Write-Through Invalidation         │
│                 │  + Proactive Warming (짧은 주기)       │
├─────────────────┼───────────────────────────────────────┤
│   NORMAL        │  SWR + TTL Jitter                     │
│   (상품, 프로필) │  + Proactive Warming (일반 주기)       │
├─────────────────┼───────────────────────────────────────┤
│   LOW           │  긴 TTL + TTL Jitter                  │
│   (정적 콘텐츠)  │  + 느슨한 Warming                     │
└─────────────────┴───────────────────────────────────────┘
```

---

## 권장 조합 (Trade-off 기반)

### 1. 일관성 우선 조합
**적합한 경우**: 금융 데이터, 재고 관리, 권한 시스템

```
분산 락 + Singleflight + Write-Through Invalidation
```

- Stampede 방지: 분산 락으로 하나의 요청만 DB 접근
- 중복 제거: Singleflight로 노드 내 요청 병합
- 즉시 갱신: 데이터 변경 시 캐시 즉시 무효화
- **Trade-off**: 락 대기로 인한 응답 지연 발생

### 2. 균형 조합
**적합한 경우**: 일반적인 비즈니스 데이터, API 응답

```
XFetch/PER + TTL Jitter + Write-Through Invalidation
```

- 확률적 조기 갱신으로 만료 전 선제 대응
- TTL 분산으로 동시 만료 방지
- 쓰기 시점 무효화로 stale 기간 최소화
- **Trade-off**: 완벽한 일관성/응답속도 둘 다 보장하지 않음

### 3. 가용성 우선 조합
**적합한 경우**: 콘텐츠 서비스, 높은 트래픽, 사용자 경험 중시

```
SWR + stale-max-age + TTL Jitter + Proactive Warming + Write-Through
```

- SWR로 즉각적인 응답 보장
- stale-max-age로 무한 stale 방지
- Proactive Warming으로 stale 발생 최소화
- Write-Through로 변경 데이터 즉시 반영
- **Trade-off**: 복잡도 증가, 일시적 stale 허용

---

## 결론

Cache Stampede 해결은 **은탄환이 없습니다**. 핵심은:

1. **데이터를 분류하라**: 모든 데이터에 같은 전략을 적용하지 말 것
2. **Trade-off를 인식하라**: 일관성과 가용성 중 무엇이 더 중요한지 판단
3. **조합을 활용하라**: 단일 솔루션보다 여러 기법 조합이 효과적
4. **쓰기 시점을 활용하라**: Write-Through Invalidation으로 stale 기간 최소화
5. **측정하고 조정하라**: 실제 트래픽 패턴에 맞게 파라미터 튜닝

**최종 권장 기본 구성**:
```
기본: TTL Jitter (모든 캐시)
  +
애플리케이션: Singleflight (노드 내 중복 제거)
  +
데이터별: CRITICAL → 분산 락
         NORMAL → SWR + stale-max-age
         LOW → 긴 TTL
  +
공통: Write-Through Invalidation (데이터 변경 시 즉시 무효화)
```

시스템의 특성과 비즈니스 요구사항에 따라 이 구성을 기반으로 조정하세요.