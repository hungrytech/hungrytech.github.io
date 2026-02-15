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

## 권장 조합

### 일반적인 상황
```
TTL Jitter + Singleflight + 분산 락
```

### 높은 가용성이 중요한 경우
```
SWR + TTL Jitter + Proactive Warming
```

### 데이터 일관성이 중요한 경우
```
분산 락 + Singleflight
```

---

## 결론

Cache Stampede는 단일 솔루션으로 해결하기보다 **여러 기법을 조합**하는 것이 효과적입니다:

1. **기본**: TTL Jitter로 만료 시점 분산
2. **애플리케이션 레벨**: Singleflight로 중복 요청 병합
3. **분산 환경**: Redis 분산 락으로 클러스터 간 조율
4. **사용자 경험**: SWR로 응답 속도 보장
5. **핫 데이터**: Proactive Warming으로 선제 대응

시스템의 특성(일관성 vs 가용성, 트래픽 패턴, 데이터 특성)에 따라 적절한 조합을 선택하세요.