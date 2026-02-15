# Redis 분산락 심화: SETNX Polling vs Pub/Sub Blocking

분산 시스템에서 공유 리소스에 대한 동시 접근을 제어하는 것은 핵심적인 문제입니다. 이 글에서는 Redis 기반 분산락의 두 가지 주요 구현 방식인 **SETNX Polling(Spin Lock)**과 **Pub/Sub Blocking**을 깊이 비교하고, 고경합(High Contention) 상황에서의 해결 전략을 제시합니다.

## 분산락의 두 가지 목적

[Martin Kleppmann](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)은 분산락을 사용하는 두 가지 목적을 명확히 구분합니다:

### 효율성(Efficiency)
중복 작업을 방지하여 비용을 절감합니다. 락이 실패해도 결과는 약간의 추가 비용(클라우드 리소스, 중복 알림 등)뿐입니다.

### 정확성(Correctness)
동시 접근으로 인한 데이터 손상을 방지합니다. 락 실패는 데이터 무결성 손상, 영구적 불일치 상태로 이어질 수 있습니다.

**이 구분이 중요한 이유**: 목적에 따라 락 구현의 복잡도와 안전성 요구 수준이 완전히 달라집니다.

---

## Redis 분산락의 기본 원리

### 원자적 락 획득

Redis 2.6.12부터 지원하는 `SET` 명령어의 옵션을 활용합니다:

```bash
SET lock_key unique_client_id NX EX 30
```

- `NX`: 키가 존재하지 않을 때만 설정 (Not eXists)
- `EX 30`: 30초 후 자동 만료

### 원자적 락 해제 (Lua 스크립트)

락 해제 시 반드시 **자신이 획득한 락만** 해제해야 합니다:

```lua
if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('del', KEYS[1])
else
    return 0
end
```

> ⚠️ **주의**: `GET`과 `DEL`을 별도로 호출하면 race condition이 발생할 수 있습니다. 반드시 Lua 스크립트로 원자성을 보장해야 합니다.

---

## 방식 1: SETNX Polling (Spin Lock)

### 동작 원리

락 획득에 실패하면 일정 시간 대기 후 재시도합니다. **Exponential Backoff with Jitter** 전략을 사용하여 충돌을 최소화합니다.

### Kotlin 구현 예시

```kotlin
import kotlinx.coroutines.delay
import kotlin.random.Random

suspend fun acquireLockWithSpinning(
    redis: RedisCommands<String, String>,
    lockKey: String,
    clientId: String,
    ttlSeconds: Long,
    maxRetries: Int = 100
): Boolean {
    var backoff = 50L // 초기 대기 시간 (ms)
    
    repeat(maxRetries) {
        val result = redis.set(
            lockKey, 
            clientId, 
            SetArgs().nx().ex(ttlSeconds)
        )
        if (result == "OK") return true

        // Exponential backoff with jitter
        val jitter = Random.nextLong(0, backoff / 2)
        delay(backoff + jitter)
        backoff = minOf(backoff * 2, 1000) // 최대 1초
    }
    return false
}

fun releaseLock(
    redis: RedisCommands<String, String>,
    lockKey: String,
    clientId: String
): Boolean {
    val script = """
        if redis.call('get', KEYS[1]) == ARGV[1] then
            return redis.call('del', KEYS[1])
        else
            return 0
        end
    """
    return redis.eval<Long>(script, listOf(lockKey), listOf(clientId)) == 1L
}
```

### 장점

| 장점 | 설명 |
|------|------|
| **단순한 구현** | 별도의 구독/발행 로직 불필요 |
| **Cluster 친화적** | Pub/Sub 브로드캐스트 오버헤드 없음 |
| **예측 가능한 네트워크** | point-to-point 쿼리만 사용 |

### 단점

| 단점 | 설명 |
|------|------|
| **CPU 낭비** | 대기 중에도 지속적으로 폴링 |
| **공정성 없음** | FIFO 순서 보장 안됨 |
| **지연 시간** | 락 해제와 획득 사이에 backoff 시간만큼 지연 |

### 권장 사용 케이스

- 고빈도 단기 락 (밀리초 단위)
- Redis Cluster 환경에서 Pub/Sub 부하가 문제될 때
- 구현 단순성이 중요할 때

---

## 방식 2: Pub/Sub Blocking

### 동작 원리

락 획득 실패 시 **채널을 구독**하고, 락 해제 시 **publish로 대기 클라이언트에 알림**합니다. Event-driven 방식으로 CPU를 효율적으로 사용합니다.

### Redisson을 활용한 Kotlin 구현

```kotlin
import org.redisson.Redisson
import org.redisson.api.RLock
import org.redisson.config.Config
import java.util.concurrent.TimeUnit

// Redisson 설정
val config = Config().apply {
    useSingleServer().address = "redis://localhost:6379"
    // Watchdog 타임아웃 설정 (기본 30초)
    lockWatchdogTimeout = 30000
}
val redisson = Redisson.create(config)

// 기본 RLock 사용 (Pub/Sub 기반)
fun executeWithLock(lockName: String, action: () -> Unit) {
    val lock: RLock = redisson.getLock(lockName)
    
    lock.lock() // Watchdog이 자동으로 만료 시간 연장
    try {
        action()
    } finally {
        lock.unlock()
    }
}

// 타임아웃과 lease time 지정
suspend fun tryExecuteWithTimeout(
    lockName: String,
    waitTime: Long,
    leaseTime: Long,
    action: suspend () -> Unit
): Boolean {
    val lock: RLock = redisson.getLock(lockName)
    
    val acquired = lock.tryLock(
        waitTime,   // 락 획득 최대 대기 시간
        leaseTime,  // 락 자동 해제 시간
        TimeUnit.SECONDS
    )
    
    if (acquired) {
        try {
            action()
        } finally {
            lock.unlock()
        }
    }
    return acquired
}
```

### Redisson의 Watchdog 메커니즘

Redisson은 **Lock Watchdog**을 통해 락 만료 문제를 해결합니다:

```
┌─────────────────────────────────────────────────────────────┐
│                      Lock Watchdog                          │
├─────────────────────────────────────────────────────────────┤
│  1. 락 획득 시 TTL = 30초 설정                               │
│  2. 매 10초마다 스케줄러가 TTL을 30초로 재설정                 │
│  3. 클라이언트 crash 시 스케줄러 중단 → 30초 후 락 자동 해제   │
└─────────────────────────────────────────────────────────────┘
```

### 장점

| 장점 | 설명 |
|------|------|
| **CPU 효율적** | 이벤트 대기 중 리소스 소모 없음 |
| **즉시 알림** | 락 해제 시 즉각적인 인지 가능 |
| **Watchdog 지원** | 장기 작업도 안전하게 처리 |

### 단점

| 단점 | 설명 |
|------|------|
| **Cluster 부하** | 모든 노드에 메시지 브로드캐스트 |
| **고빈도 부적합** | 초당 수천 개 락 시 네트워크 병목 |
| **복잡한 구현** | 채널 구독/해제 로직 필요 |

---

## 방식 3: BLPOP 하이브리드

SETNX의 단순함과 Blocking의 효율성을 결합한 접근법입니다.

### 핵심 아이디어

- **락 저장**: `lock:<name>` - SETNX로 관리
- **신호 전달**: `lock-signal:<name>` - BLPOP으로 블로킹 대기

### 왜 두 개의 키를 사용하는가?

`SET`과 `BLPOP`은 **서로 다른 Redis 자료구조**에서 동작합니다:

| 명령어 | 자료구조 | 용도 |
|--------|----------|------|
| `SET NX` | String | 락 소유권 저장 |
| `BLPOP` | List | 대기자 알림 (blocking) |

**같은 키에서 사용 불가**: `SET`으로 생성한 String 타입 키에 `BLPOP`을 호출하면 타입 오류가 발생합니다. 따라서 **별도의 키**를 사용해야 합니다.

```
┌─────────────────────────────────────────────────────────────┐
│  Key 1: "lock:order:123"         (String 타입)              │
│  ├── SET lock:order:123 client_id NX EX 30                 │
│  ├── GET lock:order:123                                    │
│  ├── DEL lock:order:123                                    │
│  └── 용도: 실제 락 상태 저장 및 소유권 확인                   │
├─────────────────────────────────────────────────────────────┤
│  Key 2: "signal:lock:order:123"  (List 타입)                │
│  ├── BLPOP signal:lock:order:123 30  (blocking 대기)        │
│  ├── LPUSH signal:lock:order:123 "1" (unlock 신호 발송)     │
│  └── 용도: 락 해제 알림 전용 (대기 → 신호 수신)              │
└─────────────────────────────────────────────────────────────┘
```

### 동작 흐름

```
Client A                              Client B
────────                              ────────
SET lock NX EX 30 → "OK" (락 획득)
                                      SET lock NX → nil (실패)
                                      BLPOP signal 30 (blocking 대기 시작)
                                           │
... 작업 수행 중 ...                        │ (대기 중, CPU 사용 없음)
                                           │
DEL lock (락 해제)                          │
LPUSH signal "1" ─────────────────────────→ BLPOP 반환 (신호 수신!)
                                      SET lock NX → "OK" (락 획득 성공)
```

### Kotlin 구현

```kotlin
class BlockingRedisLock(
    private val redis: RedisCommands<String, String>,
    private val lockKey: String,
    private val signalKey: String = "signal:$lockKey",  // 별도의 List 키
    private val ttlSeconds: Long = 30
) {
    private val clientId = UUID.randomUUID().toString()

    suspend fun <T> withLock(action: suspend () -> T): T {
        acquire()
        try {
            return action()
        } finally {
            release()
        }
    }

    private suspend fun acquire() {
        while (true) {
            // 1. SETNX로 락 획득 시도 (String 자료구조)
            val result = redis.set(
                lockKey,        // "lock:order:123"
                clientId, 
                SetArgs().nx().ex(ttlSeconds)
            )
            if (result == "OK") return

            // 2. BLPOP으로 신호 대기 (List 자료구조, blocking)
            redis.blpop(ttlSeconds.toDouble(), signalKey)  // "signal:lock:order:123"
            // 신호 수신 또는 타임아웃 후 재시도
        }
    }

    private fun release() {
        // Lua 스크립트: 락 해제(String) + 신호 발송(List)
        val script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                redis.call('del', KEYS[1])           -- String: 락 삭제
                redis.call('lpush', KEYS[2], '1')    -- List: 신호 push
                return 1
            end
            return 0
        """
        redis.eval<Long>(
            script, 
            listOf(lockKey, signalKey),  // KEYS[1]=lockKey, KEYS[2]=signalKey
            listOf(clientId)              // ARGV[1]=clientId
        )
    }
}

// 사용 예시
val lock = BlockingRedisLock(redis, "lock:order:12345")
lock.withLock {
    processOrder()
}
```

### BLPOP의 공정성 보장

> **"If multiple clients are blocked for the same key, the first client to be served is the one that has been waiting the longest."**
> — Redis Documentation

BLPOP은 자연스럽게 **FIFO 순서**를 보장합니다. 먼저 대기한 클라이언트가 먼저 신호를 받습니다.

### python-redis-lock 라이브러리의 동일한 접근법

이 패턴은 [python-redis-lock](https://github.com/ionelmc/python-redis-lock) 라이브러리에서도 동일하게 사용됩니다:

> "Uses 2 keys for each lock: `lock:<name>` - a string value for the actual lock, and `lock-signal:<name>` - a list value for signaling the waiters when the lock is released."

---

## 고경합(High Contention) 해결 전략

### Thundering Herd 문제

락이 해제되면 모든 대기자가 동시에 경쟁합니다:

```
락 해제 → 100개 클라이언트 동시 재시도 → 99개 실패 → 99개 재시도...
```

### 해결책 1: Fair Lock (공정 락)

Redisson의 Fair Lock은 **대기열**을 통해 FIFO를 보장합니다:

```kotlin
// Fair Lock - 순서대로 획득 보장
val fairLock = redisson.getFairLock("paymentProcessingLock")

fairLock.lock()
try {
    processPayment()
} finally {
    fairLock.unlock()
}
```

> ⚠️ **주의**: 대기 중인 스레드가 죽으면 5초씩 대기합니다. 5개 스레드 사망 시 25초 지연이 발생합니다.

### 해결책 2: Semaphore (동시 N개 허용)

동시 접근 수를 제한하되 완전한 배타적 락은 아닌 경우:

```kotlin
// 동시에 10개까지 허용
val semaphore = redisson.getSemaphore("dbConnectionPool")
semaphore.trySetPermits(10)

semaphore.acquire()
try {
    queryDatabase()
} finally {
    semaphore.release()
}
```

### 해결책 3: Spin Lock (Redisson)

Pub/Sub 오버헤드가 문제될 때 Exponential Backoff 기반 폴링:

```kotlin
// Pub/Sub 대신 폴링 사용
val spinLock = redisson.getSpinLock("highFrequencyLock")
spinLock.lock()
try {
    // 고빈도 짧은 작업
} finally {
    spinLock.unlock()
}
```

### 해결책 4: Request Coalescing (Singleflight)

동일 리소스에 대한 요청을 병합:

```kotlin
class Singleflight<K, V> {
    private val inFlight = ConcurrentHashMap<K, Deferred<V>>()

    suspend fun execute(key: K, loader: suspend () -> V): V {
        val existing = inFlight[key]
        if (existing != null) {
            return existing.await() // 기존 요청 결과 공유
        }

        val deferred = CompletableDeferred<V>()
        val prev = inFlight.putIfAbsent(key, deferred)
        if (prev != null) {
            return prev.await()
        }

        return try {
            val result = loader()
            deferred.complete(result)
            result
        } finally {
            inFlight.remove(key)
        }
    }
}
```

---

## SETNX vs Pub/Sub 비교표

| 기준 | SETNX Polling | Pub/Sub Blocking |
|------|---------------|------------------|
| **CPU 사용** | 높음 (polling) | 낮음 (이벤트 대기) |
| **네트워크 부하** | 낮음 (point query) | 높음 (broadcast) |
| **공정성** | 없음 | Fair Lock 지원 |
| **지연시간** | 가변적 (backoff) | 일정 (즉시 알림) |
| **구현 복잡도** | 낮음 | 높음 |
| **Cluster 적합성** | 우수 | 주의 필요 |
| **권장 상황** | 고빈도/단기 락 | 일반적 상황 |

---

## Redlock 논쟁: 안전한가?

### Martin Kleppmann의 비판

1. **Timing Assumption**: Redlock은 네트워크 지연, 프로세스 일시정지, 시계 동기화에 대한 가정에 의존합니다.

2. **Fencing Token 부재**: 
   > "Redlock does not have any facility for generating fencing tokens."

3. **Clock Drift**: Redis는 `gettimeofday`를 사용하며, 시스템 시간이 갑자기 점프할 수 있습니다.

### Antirez의 반론

1. **실용적 안전성**: 대부분의 운영 환경에서 시계는 충분히 안정적입니다.

2. **시간 검증**: Redlock은 락 획득 **전후**로 시간을 확인합니다.

### 결론

- **효율성 목적**: 단일 Redis + `SET NX EX`로 충분
- **정확성 목적**: ZooKeeper/etcd + Fencing Token 권장
- **중간 지점**: Redisson의 Watchdog 메커니즘 활용

---

## 실무 권장사항

### 상황별 선택 가이드

```
┌─────────────────────────────────────────────────────────────┐
│                    락 목적이 무엇인가?                        │
├─────────────────────┬───────────────────────────────────────┤
│    효율성           │             정확성                     │
│    (중복 방지)      │         (데이터 무결성)                 │
├─────────────────────┼───────────────────────────────────────┤
│ 단일 Redis          │ ZooKeeper/etcd                        │
│ SET NX EX           │ + Fencing Token                       │
│                     │                                       │
│ 실패해도 괜찮음      │ 실패 시 데이터 손상                    │
└─────────────────────┴───────────────────────────────────────┘
```

### 경합 수준별 선택

| 경합 수준 | 권장 방식 |
|----------|----------|
| 낮음 (초당 ~100) | Redisson RLock (Pub/Sub) |
| 중간 (초당 ~1000) | Redisson Spin Lock |
| 높음 (초당 ~10000) | 락 분할 (Sharding) + Spin Lock |
| 극한 | 락 회피 설계 (Lock-free) |

### 핵심 원칙

1. **단순함 우선**: 복잡한 솔루션은 복잡한 문제를 만듭니다
2. **목적 명확화**: 효율성인지 정확성인지 먼저 결정
3. **측정 후 최적화**: 경합이 실제로 문제인지 먼저 확인
4. **TTL 신중히 설정**: 작업 최대 시간의 2배 이상으로

---

## 참고 자료

- [How to do distributed locking — Martin Kleppmann](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- [Is Redlock safe? — Antirez](https://antirez.com/news/101)
- [Distributed Locks with Redis — Redis Documentation](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
- [Redisson Locks and Synchronizers](https://redisson.pro/docs/data-and-services/locks-and-synchronizers/)
- [Implementation Principles and Best Practices of Distributed Lock — Alibaba Cloud](https://www.alibabacloud.com/blog/implementation-principles-and-best-practices-of-distributed-lock_600811)
- [python-redis-lock — GitHub](https://github.com/ionelmc/python-redis-lock)
