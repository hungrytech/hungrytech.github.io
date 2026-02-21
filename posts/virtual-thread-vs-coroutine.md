# Virtual Thread vs Coroutine: 내부 동작 원리 비교

Java Virtual Thread와 Kotlin Coroutine은 모두 "경량 동시성"을 제공하지만, 구현 방식은 근본적으로 다릅니다. 이 글에서는 두 기술의 내부 동작 원리를 깊이 있게 비교합니다.

## 1. 왜 경량 동시성이 필요한가?

### OS Thread의 한계

```
┌─────────────────────────────────────────────────────────┐
│                    OS Thread 비용                        │
├─────────────────────────────────────────────────────────┤
│  메모리: 스택 크기 ~1MB per thread                        │
│  생성 비용: 커널 호출, 메모리 할당                         │
│  컨텍스트 스위칭: 커널 모드 전환, 캐시 무효화              │
│  동시 스레드 수: 수천 개 수준에서 한계                     │
└─────────────────────────────────────────────────────────┘
```

10,000개의 동시 요청을 처리하려면 10,000개의 OS Thread가 필요하고, 이는 약 10GB의 스택 메모리만 필요합니다. 이런 한계를 극복하기 위해 Virtual Thread와 Coroutine이 등장했습니다.

---

## 2. Java Virtual Thread 내부 구조

### 2.1 아키텍처 개요

```
┌─────────────────────────────────────────────────────────────┐
│                    Virtual Threads                           │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐                   │
│  │ VT1 │ │ VT2 │ │ VT3 │ │ VT4 │ │ ... │  (수백만 개 가능)  │
│  └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘                   │
│     │       │       │       │       │                       │
│     └───────┴───────┼───────┴───────┘                       │
│                     ▼                                        │
│  ┌─────────────────────────────────────────────────────┐    │
│  │          ForkJoinPool Scheduler (FIFO)              │    │
│  │         Work-Stealing, parallelism = CPU cores      │    │
│  └───────────────────────┬─────────────────────────────┘    │
│                          ▼                                   │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │
│  │ Carrier │ │ Carrier │ │ Carrier │ │ Carrier │           │
│  │ Thread1 │ │ Thread2 │ │ Thread3 │ │ Thread4 │           │
│  │  (OS)   │ │  (OS)   │ │  (OS)   │ │  (OS)   │           │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘           │
│              Platform Threads (CPU 코어 수만큼)              │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Continuation: Virtual Thread의 핵심

**Continuation**은 "실행 상태의 스냅샷"입니다. Virtual Thread의 모든 마법은 여기서 시작됩니다.

```java
// VirtualThread 내부 구조 (개념적)
class VirtualThread extends Thread {
    private final Continuation continuation;
    private final Executor scheduler;

    VirtualThread(Runnable task) {
        this.continuation = new VThreadContinuation(VTHREAD_SCOPE, task);
        this.scheduler = ForkJoinPool.commonPool(); // 실제로는 전용 풀
    }
}
```

#### Continuation이 저장하는 것:
- **Stack Frames**: 현재 호출 스택의 모든 프레임
- **Local Variables**: 각 프레임의 지역 변수들
- **Program Counter**: 다음 실행할 위치

```
┌─────────────────────────────────────────────────────────────┐
│                  Continuation 내부 구조                      │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                 Stack Chunk (Heap)                   │   │
│  │  ┌─────────────────────────────────────────────┐    │   │
│  │  │ Frame 3: processResult() - line 42          │    │   │
│  │  │   locals: result=..., count=5               │    │   │
│  │  ├─────────────────────────────────────────────┤    │   │
│  │  │ Frame 2: fetchData() - line 28              │    │   │
│  │  │   locals: url="...", timeout=3000           │    │   │
│  │  ├─────────────────────────────────────────────┤    │   │
│  │  │ Frame 1: handleRequest() - line 15          │    │   │
│  │  │   locals: request=..., session=...          │    │   │
│  │  └─────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  state: SUSPENDED | RUNNING                                 │
│  scope: VirtualThreadScope                                  │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 Mounting/Unmounting 메커니즘

Virtual Thread가 블로킹 작업을 만나면 자동으로 Carrier Thread에서 분리됩니다.

```
[Mounting: Virtual Thread → Carrier Thread에 연결]

1. Scheduler가 실행할 VT 선택
2. Continuation의 Stack Chunk를 Heap에서 로드
3. Carrier Thread의 스택에 프레임 복사
4. 실행 재개

┌─────────────────────────────────────────────────────────────┐
│  Heap                         │  Carrier Thread Stack       │
│  ┌─────────────────────┐      │  ┌─────────────────────┐   │
│  │ Stack Chunk (VT1)   │ ───► │  │ Frame 3            │   │
│  │   Frame 1           │      │  │ Frame 2            │   │
│  │   Frame 2           │      │  │ Frame 1            │   │
│  │   Frame 3           │      │  └─────────────────────┘   │
│  └─────────────────────┘      │          실행 중...         │
└─────────────────────────────────────────────────────────────┘
```

```
[Unmounting: Carrier Thread에서 분리]

1. I/O 블로킹 감지 (socket.read(), sleep() 등)
2. Continuation.yield() 호출
3. 현재 스택 프레임들을 Heap의 Stack Chunk로 복사
4. Carrier Thread 해제 → 다른 VT 실행 가능

┌─────────────────────────────────────────────────────────────┐
│  Carrier Thread Stack         │  Heap                       │
│  ┌─────────────────────┐      │  ┌─────────────────────┐   │
│  │ (비어 있음)         │ ◄─── │  │ Stack Chunk (VT1)   │   │
│  │                     │      │  │   Frame 1           │   │
│  │                     │      │  │   Frame 2           │   │
│  └─────────────────────┘      │  │   Frame 3           │   │
│     다른 VT 실행 가능!        │  │   state: SUSPENDED  │   │
│                               │  └─────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 2.4 블로킹 작업 감지 흐름

```java
Thread.startVirtualThread(() -> {
    // 1. VT 생성, Scheduler에 등록

    Socket socket = new Socket("server", 8080);
    // 2. 일반 코드 실행

    byte[] data = socket.getInputStream().read();
    // 3. ↑ 블로킹 호출 감지!
    //    └─ JVM 내부에서 자동으로:
    //       a) Continuation.yield() 호출
    //       b) Stack을 Heap으로 복사
    //       c) Carrier Thread 반환
    //       d) I/O 완료 대기 (epoll/kqueue)

    // 4. I/O 완료 시:
    //    a) VT를 Scheduler에 재등록
    //    b) 새 Carrier Thread에 Mount
    //    c) Continuation 재개

    process(data);
    // 5. 계속 실행
});
```

### 2.5 Pinning: Unmount가 불가능한 경우

특정 상황에서는 Virtual Thread가 Carrier Thread에서 분리되지 못합니다.

```java
// ❌ Pinning 발생 (JDK 23 이전)
synchronized (lock) {
    socket.read();  // 블로킹이지만 Unmount 불가!
    // Carrier Thread가 점유된 상태로 유지
}

// ✅ Pinning 방지
private final ReentrantLock lock = new ReentrantLock();

lock.lock();
try {
    socket.read();  // 정상적으로 Unmount
} finally {
    lock.unlock();
}
```

**JDK 24+ (JEP 491)**: `synchronized` 블록에서도 Unmount 가능하도록 개선됨!

### 2.6 스케줄러 동작

```
┌─────────────────────────────────────────────────────────────┐
│              ForkJoinPool (Work-Stealing)                    │
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │ Worker Thread 1 │  │ Worker Thread 2 │                  │
│  │ ┌─────────────┐ │  │ ┌─────────────┐ │                  │
│  │ │ Local Queue │ │  │ │ Local Queue │ │                  │
│  │ │ VT1, VT2    │ │  │ │ VT5         │ │ ◄─ Work Steal   │
│  │ └─────────────┘ │  │ └─────────────┘ │                  │
│  └─────────────────┘  └─────────────────┘                  │
│           │                    ▲                            │
│           └────────────────────┘                            │
│              큐가 비면 다른 Worker에서 훔쳐옴                 │
│                                                             │
│  Mode: FIFO (Virtual Thread용)                              │
│  Parallelism: Runtime.availableProcessors()                 │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Kotlin Coroutine 내부 구조

### 3.1 아키텍처 개요

```
┌─────────────────────────────────────────────────────────────┐
│                      Coroutines                              │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              suspend fun (소스 코드)                 │   │
│  │                       │                              │   │
│  │                       ▼ (컴파일)                     │   │
│  │              State Machine Class                     │   │
│  │         (label + Continuation 파라미터)              │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              CoroutineDispatcher                     │   │
│  │     Default | IO | Main | Unconfined                │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                Thread Pool                           │   │
│  │          (실제 코드 실행하는 스레드)                  │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 CPS (Continuation Passing Style) 변환

Kotlin 컴파일러는 모든 `suspend` 함수를 CPS로 변환합니다.

```kotlin
// 원본 코드
suspend fun fetchUser(id: String): User {
    val token = getToken()           // suspend point 1
    val user = fetchFromApi(token)   // suspend point 2
    return user
}

// 컴파일 후 (개념적)
fun fetchUser(id: String, continuation: Continuation<User>): Any? {
    // Continuation 파라미터가 추가됨
    // 반환 타입이 Any?로 변경 (User 또는 COROUTINE_SUSPENDED)
}
```

### 3.3 State Machine 생성

컴파일러는 각 suspension point를 상태(label)로 변환합니다.

```kotlin
// 원본 suspend 함수
suspend fun fetchData(): String {
    val a = fetchA()      // suspension point 1
    val b = fetchB(a)     // suspension point 2
    return a + b
}

// 컴파일 후 생성되는 State Machine (개념적 코드)
class FetchData$Continuation(
    completion: Continuation<String>
) : ContinuationImpl(completion) {

    var label: Int = 0        // 현재 상태
    var result: Any? = null   // 마지막 결과

    // 로컬 변수들 (suspension을 넘어 유지)
    var a: String? = null

    override fun invokeSuspend(result: Result<Any?>): Any? {
        this.result = result.getOrThrow()

        when (label) {
            0 -> {
                // 초기 상태
                label = 1
                val suspended = fetchA(this)  // Continuation 전달
                if (suspended === COROUTINE_SUSPENDED) {
                    return COROUTINE_SUSPENDED
                }
                // 즉시 반환된 경우 fall-through
            }
            1 -> {
                // fetchA() 완료 후
                a = result as String
                label = 2
                val suspended = fetchB(a, this)
                if (suspended === COROUTINE_SUSPENDED) {
                    return COROUTINE_SUSPENDED
                }
            }
            2 -> {
                // fetchB() 완료 후
                val b = result as String
                return a + b  // 최종 결과 반환
            }
        }
    }
}
```

### 3.4 State Machine 시각화

```
┌─────────────────────────────────────────────────────────────┐
│                    State Machine 구조                        │
│                                                             │
│  ┌─────────┐      ┌─────────┐      ┌─────────┐            │
│  │ label=0 │─────►│ label=1 │─────►│ label=2 │            │
│  │ 시작     │      │ fetchA  │      │ fetchB  │───► 완료   │
│  │         │      │ 완료    │      │ 완료    │            │
│  └─────────┘      └─────────┘      └─────────┘            │
│       │                │                │                  │
│       ▼                ▼                ▼                  │
│  fetchA(cont)    fetchB(a, cont)    return a+b           │
│       │                │                                   │
│       └────────────────┴─────────────────────────────────  │
│              COROUTINE_SUSPENDED 반환 시                    │
│              → 실행 중단, 나중에 resumeWith()로 재개        │
└─────────────────────────────────────────────────────────────┘
```

### 3.5 Continuation 인터페이스

```kotlin
interface Continuation<in T> {
    val context: CoroutineContext

    fun resumeWith(result: Result<T>)
}

// 사용 예시
suspend fun <T> suspendCoroutine(
    block: (Continuation<T>) -> Unit
): T = suspendCoroutineUninterceptedOrReturn { cont ->
    block(cont)
    COROUTINE_SUSPENDED
}
```

### 3.6 COROUTINE_SUSPENDED의 의미

```kotlin
internal val COROUTINE_SUSPENDED = Any()

// suspend 함수 호출 결과:
// 1. 실제 결과 T 반환 → 중단 없이 계속 실행
// 2. COROUTINE_SUSPENDED 반환 → 실제로 중단됨

suspend fun example(): Int {
    val result = delay(1000)  // COROUTINE_SUSPENDED 반환
    // ↑ 여기서 실행 중단
    // 1초 후 continuation.resumeWith(Unit) 호출
    // ↓ 여기서 재개
    return 42
}
```

### 3.7 Dispatcher 동작

```
┌─────────────────────────────────────────────────────────────┐
│                    Dispatchers 비교                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Dispatchers.Default                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Thread Pool Size: CPU 코어 수                        │   │
│  │ 용도: CPU 집약적 작업                                │   │
│  │ 특징: Work-stealing, 공유 풀                         │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Dispatchers.IO                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Thread Pool Size: max(64, CPU 코어 수)               │   │
│  │ 용도: I/O 블로킹 작업                                │   │
│  │ 특징: Default와 스레드 공유, 탄력적 확장              │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Dispatchers.Main                                           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Thread: UI 스레드 (Android, JavaFX 등)               │   │
│  │ 용도: UI 업데이트                                    │   │
│  │ 특징: 단일 스레드, 플랫폼별 구현                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Dispatchers.Unconfined                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Thread: 호출한 스레드에서 시작                       │   │
│  │ 용도: 테스트, 특수 케이스                            │   │
│  │ 특징: 첫 suspension까지만, 이후 resume한 스레드      │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. 핵심 차이점 비교

### 4.1 구현 레벨

```
┌─────────────────────────────────────────────────────────────┐
│                     Virtual Thread                           │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    JVM Level                         │   │
│  │  • java.lang.Thread의 새로운 구현                    │   │
│  │  • JVM 내부에서 Continuation 지원                    │   │
│  │  • 기존 Thread API 100% 호환                         │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                       Coroutine                              │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Compiler + Library Level                │   │
│  │  • 컴파일러가 State Machine으로 변환                 │   │
│  │  • kotlinx.coroutines 라이브러리                     │   │
│  │  • suspend 키워드로 명시적 마킹 필요                 │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 스택 관리 비교

```
┌─────────────────────────────────────────────────────────────┐
│                Virtual Thread Stack                          │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Stack Chunk (Heap 저장)                 │   │
│  │                                                      │   │
│  │  • 실제 JVM 스택 프레임 구조                         │   │
│  │  • 동적 크기 조절 (필요에 따라 grow/shrink)          │   │
│  │  • GC가 관리하는 힙 메모리                          │   │
│  │  • 초기 ~1KB, 필요시 확장                           │   │
│  │                                                      │   │
│  │  ┌────────────────────────────────────────────┐     │   │
│  │  │ Stack Chunk Object                         │     │   │
│  │  │  ├─ Frame: method1() locals, PC            │     │   │
│  │  │  ├─ Frame: method2() locals, PC            │     │   │
│  │  │  └─ Frame: method3() locals, PC            │     │   │
│  │  └────────────────────────────────────────────┘     │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                 Coroutine State Machine                      │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Generated Class (Heap 저장)             │   │
│  │                                                      │   │
│  │  • 컴파일러가 생성한 상태 머신 객체                   │   │
│  │  • label(상태) + 필드(로컬 변수)                     │   │
│  │  • 스택 프레임 개념 없음                             │   │
│  │  • 매우 가벼움 (~수십 바이트)                        │   │
│  │                                                      │   │
│  │  ┌────────────────────────────────────────────┐     │   │
│  │  │ Continuation Object                        │     │   │
│  │  │  ├─ label: Int = 2                         │     │   │
│  │  │  ├─ localVar1: String = "..."              │     │   │
│  │  │  ├─ localVar2: Int = 42                    │     │   │
│  │  │  └─ completion: Continuation<T>            │     │   │
│  │  └────────────────────────────────────────────┘     │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 블로킹 처리 방식

| 관점 | Virtual Thread | Coroutine |
|------|----------------|-----------|
| **블로킹 감지** | JVM이 자동 감지 | 명시적 suspend 필요 |
| **기존 블로킹 API** | 그대로 사용 가능 | withContext(IO) 필요 |
| **논블로킹 강제** | X (투명하게 처리) | O (suspend 전파) |

```java
// Virtual Thread: 기존 블로킹 코드 그대로 사용
Thread.startVirtualThread(() -> {
    Socket socket = new Socket("server", 80);
    socket.getInputStream().read();  // 자동으로 unmount/mount
});
```

```kotlin
// Coroutine: suspend 함수 사용 필요
suspend fun fetchData() {
    // ❌ 블로킹 - 스레드 점유
    val data = URL("...").readText()

    // ✅ suspend - 비동기
    val data = httpClient.get("...").body()
}
```

### 4.4 컨텍스트 스위칭 비용

```
┌─────────────────────────────────────────────────────────────┐
│              Virtual Thread Context Switch                   │
│                                                             │
│  비용 요소:                                                  │
│  1. Stack Chunk를 Heap에서 읽기/쓰기                        │
│  2. Stack Frame들의 메모리 복사                              │
│  3. Carrier Thread 스케줄링                                 │
│                                                             │
│  특징:                                                       │
│  • 깊은 호출 스택 → 더 많은 복사 비용                        │
│  • 하지만 OS 컨텍스트 스위칭보다 훨씬 저렴                   │
│  • 수천 배 더 많은 동시 "스레드" 가능                        │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│               Coroutine Context Switch                       │
│                                                             │
│  비용 요소:                                                  │
│  1. State Machine의 label 변경                              │
│  2. resumeWith() 호출                                       │
│  3. Dispatcher에 따른 스레드 전환 (있을 경우)                │
│                                                             │
│  특징:                                                       │
│  • 호출 스택 깊이와 무관 (state로 평탄화됨)                  │
│  • 매우 가벼운 상태 전이                                     │
│  • 같은 Dispatcher면 스레드 전환 없을 수 있음                │
└─────────────────────────────────────────────────────────────┘
```

### 4.5 종합 비교표

| 관점 | Virtual Thread | Coroutine |
|------|----------------|-----------|
| **구현 레벨** | JVM (java.lang.Thread) | 컴파일러 + 라이브러리 |
| **스택 구조** | 실제 스택 (Heap 저장) | 상태 머신 객체 |
| **메모리 (개당)** | ~1KB (동적) | ~수십 바이트 |
| **블로킹 처리** | 자동 unmount | 명시적 suspend |
| **기존 코드 호환** | 100% 호환 | suspend 전파 필요 |
| **스케줄링** | ForkJoinPool (JVM) | Dispatcher (라이브러리) |
| **Structured Concurrency** | 미지원 (별도 API) | 내장 지원 |
| **취소 처리** | interrupt() | Job.cancel() |
| **디버깅** | 일반 스레드 덤프 | 특수 도구 필요 |

---

## 5. 메모리 구조 상세 비교

### 5.1 Virtual Thread 메모리 레이아웃

```
┌─────────────────────────────────────────────────────────────┐
│                 VirtualThread Object                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ tid: long                          (8 bytes)        │   │
│  │ name: String                       (reference)      │   │
│  │ state: int                         (4 bytes)        │   │
│  │ carrierThread: Thread              (reference)      │   │
│  │ continuation: Continuation         (reference)      │   │
│  │ scheduler: Executor                (reference)      │   │
│  │ parkPermit: boolean                (1 byte)         │   │
│  │ ...                                                 │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │            Continuation (VThreadContinuation)        │   │
│  │  ┌───────────────────────────────────────────────┐  │   │
│  │  │ scope: ContinuationScope        (reference)   │  │   │
│  │  │ target: Runnable                (reference)   │  │   │
│  │  │ parent: Continuation            (reference)   │  │   │
│  │  │ stackChunk: StackChunk          (reference)   │  │   │
│  │  │ mounted: boolean                (1 byte)      │  │   │
│  │  └───────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    StackChunk                        │   │
│  │  • GC managed heap object                           │   │
│  │  • 동적 크기 (초기 ~1KB)                             │   │
│  │  • 여러 chunk가 linked list로 연결 가능              │   │
│  │                                                      │   │
│  │  ┌───────────────────────────────────────────────┐  │   │
│  │  │ [Frame 1][Frame 2][Frame 3]...[Frame N]       │  │   │
│  │  │    │         │         │           │          │  │   │
│  │  │    ▼         ▼         ▼           ▼          │  │   │
│  │  │  locals   locals   locals      locals         │  │   │
│  │  │  PC       PC       PC          PC             │  │   │
│  │  └───────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Coroutine 메모리 레이아웃

```
┌─────────────────────────────────────────────────────────────┐
│          Generated State Machine Class Instance              │
│                                                             │
│  class FetchData$suspendImpl$1 : ContinuationImpl {         │
│    ┌─────────────────────────────────────────────────────┐ │
│    │ label: Int = 0                    (4 bytes)         │ │
│    │ result: Any? = null               (reference)       │ │
│    │                                                     │ │
│    │ // 로컬 변수들 (suspension 넘어 유지해야 하는 것들)   │ │
│    │ $a: String? = null                (reference)       │ │
│    │ $b: Int = 0                       (4 bytes)         │ │
│    │                                                     │ │
│    │ // 상위 Continuation 참조                           │ │
│    │ completion: Continuation<T>       (reference)       │ │
│    └─────────────────────────────────────────────────────┘ │
│  }                                                          │
│                                                             │
│  총 크기: 객체 헤더(~16 bytes) + 필드들(~40 bytes)          │
│         ≈ 수십 바이트                                       │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 메모리 사용량 비교

```
┌─────────────────────────────────────────────────────────────┐
│                 100만 개 동시 실행 시                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Platform Threads (OS)                                      │
│  └─ 1,000,000 × 1MB = ~1TB (불가능)                         │
│                                                             │
│  Virtual Threads                                            │
│  └─ 1,000,000 × ~1KB = ~1GB                                 │
│     (실제로는 스택 깊이에 따라 다름)                         │
│                                                             │
│  Coroutines                                                 │
│  └─ 1,000,000 × ~100 bytes = ~100MB                         │
│     (State Machine 객체 크기에 따라)                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. 실제 동작 예시

### 6.1 Virtual Thread I/O 흐름

```java
// 코드
Thread vt = Thread.startVirtualThread(() -> {
    System.out.println("1. 시작");
    try {
        Thread.sleep(1000);  // 블로킹 호출
    } catch (InterruptedException e) {}
    System.out.println("2. 완료");
});

// 내부 동작 흐름
/*
┌─────────────────────────────────────────────────────────────┐
│ T=0ms                                                       │
│  [Carrier-1] ◄── VT1 mounted                                │
│              실행: "1. 시작" 출력                            │
│              Thread.sleep(1000) 호출                        │
│                    │                                        │
│                    ▼                                        │
│              JVM이 블로킹 감지!                              │
│              Continuation.yield()                           │
│              Stack → Heap 복사                              │
│              VT1 unmounted                                  │
│  [Carrier-1] ◄── (비어있음, 다른 VT 실행 가능)              │
├─────────────────────────────────────────────────────────────┤
│ T=1000ms                                                    │
│  타이머 만료!                                                │
│  VT1을 Scheduler에 재등록                                   │
│  [Carrier-2] ◄── VT1 mounted (다른 Carrier일 수 있음)       │
│              Heap → Stack 복원                              │
│              실행 재개: "2. 완료" 출력                       │
│              VT1 종료, unmounted                            │
└─────────────────────────────────────────────────────────────┘
*/
```

### 6.2 Coroutine Suspend 흐름

```kotlin
// 코드
suspend fun process() {
    println("1. 시작")
    delay(1000)  // suspend point
    println("2. 완료")
}

// 컴파일 후 State Machine
/*
┌─────────────────────────────────────────────────────────────┐
│ invokeSuspend(result) 호출                                  │
├─────────────────────────────────────────────────────────────┤
│ label=0:                                                    │
│   println("1. 시작")                                        │
│   label = 1                                                 │
│   val suspended = delay(1000, this)  // Continuation 전달   │
│   if (suspended === COROUTINE_SUSPENDED) {                  │
│       return COROUTINE_SUSPENDED     // 여기서 반환!        │
│   }                                                         │
│   // fall-through (즉시 완료된 경우)                         │
├─────────────────────────────────────────────────────────────┤
│ ... 1초 후 타이머가 continuation.resumeWith(Unit) 호출 ...  │
├─────────────────────────────────────────────────────────────┤
│ label=1:                                                    │
│   println("2. 완료")                                        │
│   return Unit                        // 코루틴 완료         │
└─────────────────────────────────────────────────────────────┘
*/
```

---

## 7. 성능 특성

### 7.1 생성 비용

| 항목 | Platform Thread | Virtual Thread | Coroutine |
|------|----------------|----------------|-----------|
| 생성 시간 | ~1ms | ~1μs | ~0.1μs |
| 메모리 할당 | ~1MB (OS) | ~1KB (Heap) | ~100B (Heap) |
| 커널 호출 | O | X | X |

### 7.2 컨텍스트 스위칭

| 항목 | Platform Thread | Virtual Thread | Coroutine |
|------|----------------|----------------|-----------|
| 스위칭 비용 | ~1-10μs | ~0.1-1μs | ~0.01-0.1μs |
| 커널 모드 전환 | O | X | X |
| 캐시 영향 | 높음 | 낮음 | 매우 낮음 |

### 7.3 대량 동시 실행

```
동시 실행 수 vs 메모리 사용량

동시 수    | Platform Thread | Virtual Thread | Coroutine
-----------|-----------------|----------------|------------
1,000      | 1GB            | 1MB            | 100KB
10,000     | 10GB           | 10MB           | 1MB
100,000    | 100GB (X)      | 100MB          | 10MB
1,000,000  | 불가능          | 1GB            | 100MB
```

---

## 8. 선택 기준

### 8.1 Virtual Thread를 선택해야 할 때

```
✅ 기존 Java 코드베이스가 크고 마이그레이션 비용이 높을 때
✅ 기존 블로킹 라이브러리를 그대로 사용해야 할 때
✅ 팀이 Thread API에 익숙할 때
✅ JDBC, 파일 I/O 등 블로킹 API를 많이 사용할 때
✅ 간단한 동시성 모델이 필요할 때
```

### 8.2 Coroutine을 선택해야 할 때

```
✅ Kotlin을 주력으로 사용할 때
✅ Structured Concurrency가 중요할 때
✅ 세밀한 동시성 제어가 필요할 때 (취소, 타임아웃 등)
✅ 리액티브 스트림과의 통합이 필요할 때
✅ Android 개발을 할 때
✅ 메모리 효율이 최우선일 때
```

### 8.3 함께 사용하기

```kotlin
// Kotlin에서 Virtual Thread 사용
val vtDispatcher = Executors.newVirtualThreadPerTaskExecutor()
    .asCoroutineDispatcher()

suspend fun hybridApproach() = withContext(vtDispatcher) {
    // Coroutine의 구조화된 동시성 +
    // Virtual Thread의 블로킹 API 호환성

    val connection = DriverManager.getConnection(url)  // 블로킹 OK
    val result = connection.executeQuery(sql)
    // ...
}
```

---

## 9. 정리

| 관점 | Virtual Thread | Coroutine |
|------|----------------|-----------|
| **철학** | "기존 코드를 바꾸지 마라" | "비동기를 명시적으로 표현하라" |
| **핵심 메커니즘** | JVM Continuation | 컴파일러 State Machine |
| **스택** | 실제 스택 (Heap 저장) | 상태 머신 객체 |
| **장점** | 호환성, 단순함 | 경량성, 구조화 |
| **단점** | 메모리 사용량 | 학습 곡선, 전파 필요 |

두 기술 모두 "M:N 스케줄링"을 통해 소수의 OS 스레드로 대량의 동시 작업을 처리합니다. 차이는 **구현 방식**과 **프로그래밍 모델**에 있습니다.

- **Virtual Thread**: JVM이 마법을 부려 기존 코드가 자동으로 경량화
- **Coroutine**: 컴파일러와 개발자가 협력하여 명시적으로 경량화

---

## 참고 자료

- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
- [JEP 491: Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491)
- [The Basis of Virtual Threads: Continuations](https://foojay.io/today/the-basis-of-virtual-threads-continuations/)
- [Kotlin KEEP: Coroutines Proposal](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md)
- [Inside Kotlin Coroutines: State Machines](https://proandroiddev.com/inside-kotlin-coroutines-state-machines-continuations-and-structured-concurrency-b8d3d4e48e62)
