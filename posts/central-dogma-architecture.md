# Central Dogma 아키텍처 분석

Central Dogma는 LINE에서 개발한 Git 기반의 중앙집중식 설정 저장소로, **서버 재배포 없이 실시간으로 설정 변경을 반영**할 수 있게 해주는 라이브러리입니다.

## 1. 핵심 개념

### 1.1 기본 구조

```
┌─────────────────────────────────────────────────────────────┐
│                    Central Dogma Server                      │
│  ┌─────────────────┐    ┌─────────────────────────────────┐ │
│  │   Git Storage   │    │         Watch Service           │ │
│  │  (JGit/RocksDB) │◄───│  (Long Polling / CompletableFuture) │
│  └────────┬────────┘    └─────────────────────────────────┘ │
│           │                           ▲                      │
│           ▼                           │                      │
│  ┌─────────────────┐                  │                      │
│  │ CommitIdDatabase│    ┌─────────────┴───────────┐         │
│  │ (Revision→SHA1) │    │     CommitWatchers      │         │
│  └─────────────────┘    │ (PathPattern → Watch[]) │         │
│                         └─────────────────────────┘         │
└─────────────────────────────────────────────────────────────┘
                              ▲
                              │ HTTP/2 (gRPC)
                              │ Long Polling with If-None-Match
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     Application (Client)                     │
│  ┌─────────────────┐    ┌─────────────────────────────────┐ │
│  │     Watcher     │───►│        Config Cache             │ │
│  │ (AbstractWatcher)│    │   (Latest Revision Data)       │ │
│  └─────────────────┘    └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 서버 재배포 없이 설정 변경이 가능한 이유

1. **Git 기반 버전 관리**: 모든 설정이 Git 리포지토리에 저장되어 버전 관리됨
2. **Long Polling Watch**: 클라이언트가 서버에 변경 감지 요청을 보내고 대기
3. **즉시 알림**: 설정 변경 시 대기 중인 모든 클라이언트에게 즉시 알림
4. **클라이언트 캐시 갱신**: 알림 받은 클라이언트가 새 설정을 가져와 메모리 캐시 갱신

## 2. Watch 메커니즘 (핵심)

### 2.1 클라이언트 측 Watcher

**파일: `client/java/src/main/java/com/linecorp/centraldogma/client/Watcher.java`**

```java
public interface Watcher<T> extends AutoCloseable {
    // 현재 최신 값 조회
    Latest<T> latest();

    // 최신 값의 CompletableFuture 반환
    CompletableFuture<Latest<T>> latestValue();

    // 초기값 로드까지 대기
    void awaitInitialValue(long timeout, TimeUnit unit) throws Exception;

    // 변경 리스너 등록
    void watch(BiConsumer<? super Revision, ? super T> listener);
}
```

**파일: `client/java/src/main/java/com/linecorp/centraldogma/client/AbstractWatcher.java`**

핵심 메커니즘은 `scheduleWatch()` 메서드에 있습니다:

```java
private void scheduleWatch(int numAttemptsSoFar) {
    if (isClosed()) {
        return;
    }

    // 지수 백오프 적용
    final long delay = watchScheduler.delay(numAttemptsSoFar, TimeUnit.MILLISECONDS);
    watchFuture = executor.schedule(() -> doWatch(numAttemptsSoFar), delay, TimeUnit.MILLISECONDS);
}
```

동작 흐름:
1. `scheduleWatch(0)` 호출로 Watch 시작
2. `doWatch()` 실행 - 서버에 Long Polling 요청
3. 변경 감지 시 `notifyListeners()` 호출
4. 다시 `scheduleWatch(0)` 호출하여 다음 변경 대기
5. 실패 시 `numAttemptsSoFar`를 증가시켜 지수 백오프 적용

### 2.2 서버 측 Watch 처리

#### Watch 요청 수신

**파일: `server/src/main/java/com/linecorp/centraldogma/server/internal/api/converter/WatchRequestConverter.java`**

```java
// HTTP 헤더에서 Watch 정보 추출
public WatchRequest convertRequest() {
    // If-None-Match 헤더에서 마지막으로 알고 있는 리비전 추출
    String ifNoneMatch = req.headers().get(HttpHeaderNames.IF_NONE_MATCH);
    Revision lastKnownRevision = extractRevision(ifNoneMatch);

    // Prefer 헤더에서 타임아웃 추출 (기본 120초)
    long timeoutMillis = timeoutMillis(req.headers().get(HttpHeaderNames.PREFER));

    return new WatchRequest(lastKnownRevision, timeoutMillis, notifyEntryNotFound);
}
```

#### Watch 등록 및 관리

**파일: `server/src/main/java/com/linecorp/centraldogma/server/internal/storage/repository/git/GitRepository.java`**

```java
// 라인 965-994
public CompletableFuture<Revision> watch(Revision lastKnownRevision, String pathPattern) {
    // 1. 현재 이미 변경된 것이 있는지 확인
    Revision latestRevision = blockingFindLatestRevision(lastKnownRevision, pathPattern);

    if (latestRevision != null) {
        // 이미 변경이 있으면 즉시 반환
        return CompletableFuture.completedFuture(latestRevision);
    }

    // 2. 변경이 없으면 Watcher 등록하고 대기
    CompletableFuture<Revision> future = new CompletableFuture<>();
    commitWatchers.add(lastKnownRevision, pathPattern, future);
    return future;
}
```

**파일: `server/src/main/java/com/linecorp/centraldogma/server/internal/storage/repository/git/CommitWatchers.java`**

```java
// Watch 등록 (라인 48-80)
public void add(Revision lastKnownRev, String pathPattern, CompletableFuture<Revision> future) {
    PathPatternFilter filter = PathPatternFilter.of(pathPattern);
    Watch watch = new Watch(lastKnownRev, future);

    synchronized (watchesMap) {
        watchesMap.computeIfAbsent(filter, k -> new HashSet<>()).add(watch);
    }

    // Future 완료 시 자동 정리
    future.whenComplete((revision, error) -> {
        synchronized (watchesMap) {
            Set<Watch> watches = watchesMap.get(filter);
            if (watches != null) {
                watches.remove(watch);
            }
        }
    });
}
```

### 2.3 변경 알림 흐름

```
설정 파일 변경 (Commit)
         │
         ▼
┌─────────────────────────────────────────┐
│       CommitExecutor.execute()          │
│  1. Write Lock 획득                      │
│  2. Git Commit 생성                      │
│  3. CommitIdDatabase에 Revision 저장     │
│  4. Write Lock 해제                      │
│  5. notifyWatchers() 호출 ◄─── 락 해제 후 │
└─────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│    GitRepository.notifyWatchers()       │
│  (라인 1053-1067)                        │
│                                         │
│  for (DiffEntry diff : diffEntries) {   │
│      switch (diff.changeType) {         │
│          case ADD, MODIFY:              │
│              notify(revision, path);    │
│          case DELETE:                   │
│              notify(revision, oldPath); │
│      }                                  │
│  }                                      │
└─────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│      CommitWatchers.notify()            │
│  (라인 82-123)                          │
│                                         │
│  synchronized (watchesMap) {            │
│      for (PathPatternFilter filter :    │
│           watchesMap.keySet()) {        │
│          if (filter.matches(path)) {    │
│              // 해당 패턴 구독자들 추출   │
│              eligibleWatches.addAll(...);│
│          }                              │
│      }                                  │
│  }                                      │
│                                         │
│  // 락 밖에서 알림 전송                   │
│  for (Watch watch : eligibleWatches) {  │
│      watch.future.complete(revision);   │
│  }                                      │
└─────────────────────────────────────────┘
         │
         ▼
클라이언트의 CompletableFuture 완료
         │
         ▼
클라이언트가 새 설정값 조회 → 메모리 캐시 갱신
```

## 3. Git 저장소 구조

### 3.1 Revision 관리

**파일: `server/src/main/java/com/linecorp/centraldogma/server/internal/storage/repository/git/DefaultCommitIdDatabase.java`**

Central Dogma는 Git의 SHA1 해시 대신 순차적인 정수 리비전을 사용합니다:

```
commit_ids.dat 파일 구조:
┌──────────────────────────────────────────────┐
│ Record 1: [Revision 1 (4bytes)][SHA1 (20bytes)] │
│ Record 2: [Revision 2 (4bytes)][SHA1 (20bytes)] │
│ Record 3: [Revision 3 (4bytes)][SHA1 (20bytes)] │
│ ...                                           │
└──────────────────────────────────────────────┘

- 레코드 크기: 24바이트 고정
- O(1) 조회: offset = (revision - 1) * 24
- Append-only 방식으로 새 리비전 추가
```

### 3.2 저장소 옵션

1. **JGit (기본)**: 파일 시스템 기반 Git 저장소
2. **RocksDB**: 고성능 키-값 저장소 기반 (암호화 지원)

**파일: `server/src/main/java/com/linecorp/centraldogma/server/internal/storage/repository/git/rocksdb/RocksDbRepository.java`**

## 4. 경로 패턴 매칭

**파일: `server/src/main/java/com/linecorp/centraldogma/server/internal/storage/repository/git/PathPatternFilter.java`**

```java
// 지원되는 패턴 형식
"config.json"           → "/**/config.json" (자동 와일드카드 접두사)
"/**/*.json"            → 모든 JSON 파일
"app/settings/**"       → 특정 디렉토리 하위 전체
"path1.json,path2.yaml" → 쉼표로 구분된 복수 패턴
```

패턴 컴파일 결과는 ThreadLocal LRU 캐시에 저장되어 성능 최적화됩니다.

## 5. 타임아웃 및 에러 처리

### 5.1 Watch 타임아웃

**파일: `server/src/main/java/com/linecorp/centraldogma/server/internal/api/WatchService.java`**

```java
// 라인 140-178
private void scheduleTimeout(CompletableFuture<?> result, long timeoutMillis) {
    // Thundering Herd 방지를 위한 지터 적용 (20%)
    long jitteredTimeout = applyJitter(timeoutMillis, JITTER_RATE);

    ScheduledFuture<?> timeoutFuture = eventLoop.schedule(() -> {
        result.completeExceptionally(new CancellationException("watch timeout"));
    }, jitteredTimeout, TimeUnit.MILLISECONDS);

    // 정상 완료 시 타임아웃 취소
    result.whenComplete((r, e) -> timeoutFuture.cancel(false));
}
```

### 5.2 클라이언트 지수 백오프

실패 시 클라이언트는 지수 백오프를 적용하여 재시도:

```
시도 1: 즉시
시도 2: 200ms 대기
시도 3: 400ms 대기
시도 4: 800ms 대기
...
최대 대기: 10초
```

## 6. 동시성 제어

### 6.1 Read-Write Lock 전략

**파일: `server/src/main/java/com/linecorp/centraldogma/server/internal/storage/repository/git/GitRepository.java`**

```java
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();

// 읽기 작업 (동시 가능)
readLock: blockingFind(), watch(), findLatestRevision()

// 쓰기 작업 (배타적)
writeLock: commit(), setHeadRevision()

// 중요: 알림은 락 해제 후 수행 (데드락 방지)
notifyWatchers(): 락 없이 실행
```

### 6.2 Watch Map 동기화

```java
// CommitWatchers 내부
Map<PathPatternFilter, Set<Watch>> watchesMap;

synchronized (watchesMap) {
    // Watch 등록/제거/알림 모두 동기화
}
```

## 7. 성능 최적화

| 최적화 기법 | 설명 |
|------------|------|
| **경로 패턴 캐싱** | ThreadLocal LRU 캐시 (512개 필터) |
| **O(1) 리비전 조회** | 고정 크기 바이너리 파일 |
| **지터 타임아웃** | Thundering Herd 방지 |
| **Watch Map LRU** | 최대 8192개 패턴으로 메모리 제한 |
| **락 외부 알림** | 데드락 방지 및 처리량 향상 |

## 8. 사용 예제

### 8.1 클라이언트 코드

```java
// Central Dogma 클라이언트 생성
CentralDogma dogma = CentralDogma.forHost("centraldogma.example.com", 36462);

// 설정 파일 Watcher 생성
Watcher<JsonNode> watcher = dogma.forRepo("project", "repo")
    .watcher(Query.ofJsonPath("/config.json", "$.database"))
    .build();

// 리스너 등록 - 변경 시 자동 호출
watcher.watch((revision, config) -> {
    System.out.println("Config updated to revision " + revision);
    updateDatabaseConnection(config);
});

// 초기값 대기 (서버 시작 시)
watcher.awaitInitialValue(10, TimeUnit.SECONDS);

// 현재 값 조회
JsonNode currentConfig = watcher.latest().value();
```

### 8.2 설정 변경 (API)

```bash
# 설정 파일 업데이트
curl -X POST "http://centraldogma:36462/api/v1/projects/project/repos/repo/contents" \
  -H "Content-Type: application/json" \
  -d '{
    "commitMessage": {"summary": "Update database config"},
    "changes": [{
      "path": "/config.json",
      "type": "UPSERT_JSON",
      "content": {"database": {"host": "newhost", "port": 5432}}
    }]
  }'
```

## 9. 아키텍처 장점

1. **실시간성**: Long Polling으로 거의 즉각적인 설정 반영
2. **버전 관리**: Git 기반으로 변경 이력 추적 및 롤백 가능
3. **효율성**: Watch 패턴으로 필요한 설정만 구독
4. **신뢰성**: CompletableFuture 기반의 안정적인 비동기 처리
5. **확장성**: 수천 개의 동시 Watch 처리 가능

## 10. Spring Boot 통합

### 10.1 의존성 추가

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.linecorp.centraldogma</groupId>
    <artifactId>centraldogma-client-spring-boot3-starter</artifactId>
    <version>0.68.0</version>
</dependency>
```

```groovy
// build.gradle
implementation 'com.linecorp.centraldogma:centraldogma-client-spring-boot3-starter:0.68.0'
```

### 10.2 application.yml 설정

```yaml
centraldogma:
  hosts:
    - centraldogma.example.com:36462
  access-token: appToken-cffed349-d573-457f-8f74-4727ad9341ce
  health-check-interval-millis: 15000
  initialization-timeout-millis: 10000
  use-tls: false
```

### 10.3 실시간 설정 변경 예제

#### 예제 1: 데이터베이스 연결 정보 동적 변경

**설정 DTO**

```java
@Data
public class DatabaseConfig {
    private String host;
    private int port;
    private String username;
    private String password;
    private int maxPoolSize;
}
```

**동적 설정 관리 서비스**

```java
@Service
@Slf4j
public class DynamicConfigService {

    private final CentralDogma centralDogma;
    private final ObjectMapper objectMapper;

    private volatile DatabaseConfig databaseConfig;
    private Watcher<JsonNode> watcher;

    public DynamicConfigService(CentralDogma centralDogma, ObjectMapper objectMapper) {
        this.centralDogma = centralDogma;
        this.objectMapper = objectMapper;
    }

    @PostConstruct
    public void init() throws Exception {
        // Watcher 생성 및 시작
        watcher = centralDogma.forRepo("my-project", "config")
            .watcher(Query.ofJsonPath("/database.json"))
            .start();

        // 변경 리스너 등록
        watcher.watch((revision, jsonNode) -> {
            log.info("Config updated to revision: {}", revision);
            updateConfig(jsonNode);
        });

        // 초기값 로드 대기 (최대 10초)
        watcher.awaitInitialValue(10, TimeUnit.SECONDS);

        // 초기 설정 적용
        updateConfig(watcher.latest().value());
        log.info("Initial config loaded: {}", databaseConfig);
    }

    private void updateConfig(JsonNode jsonNode) {
        try {
            this.databaseConfig = objectMapper.treeToValue(jsonNode, DatabaseConfig.class);
            // 필요시 DataSource 재구성 등 추가 작업
            onConfigChanged(databaseConfig);
        } catch (JsonProcessingException e) {
            log.error("Failed to parse config", e);
        }
    }

    private void onConfigChanged(DatabaseConfig newConfig) {
        // 설정 변경 시 수행할 작업
        // 예: HikariCP 풀 재구성, 캐시 갱신 등
        log.info("Applying new database config: host={}, port={}",
                 newConfig.getHost(), newConfig.getPort());
    }

    public DatabaseConfig getDatabaseConfig() {
        return databaseConfig;
    }

    @PreDestroy
    public void destroy() {
        if (watcher != null) {
            watcher.close();
        }
    }
}
```

#### 예제 2: Feature Flag 실시간 토글

**Feature Flag 설정**

```java
@Data
public class FeatureFlags {
    private boolean newCheckoutEnabled;
    private boolean darkModeEnabled;
    private int maxUploadSizeMb;
    private List<String> betaUserIds;
}
```

**Feature Flag 서비스**

```java
@Service
@Slf4j
public class FeatureFlagService {

    private final AtomicReference<FeatureFlags> flags = new AtomicReference<>(new FeatureFlags());
    private Watcher<FeatureFlags> watcher;

    public FeatureFlagService(CentralDogma centralDogma) throws Exception {
        // map()을 사용하여 JsonNode → FeatureFlags 변환
        this.watcher = centralDogma.forRepo("my-project", "config")
            .watcher(Query.ofJsonPath("/features.json"))
            .map(jsonNode -> {
                ObjectMapper mapper = new ObjectMapper();
                try {
                    return mapper.treeToValue(jsonNode, FeatureFlags.class);
                } catch (JsonProcessingException e) {
                    throw new RuntimeException(e);
                }
            })
            .start();

        // 변경 시 AtomicReference 갱신
        watcher.watch((revision, newFlags) -> {
            log.info("Feature flags updated at revision {}", revision);
            flags.set(newFlags);
        });

        watcher.awaitInitialValue(10, TimeUnit.SECONDS);
        flags.set(watcher.latest().value());
    }

    // 서비스 코드에서 사용
    public boolean isNewCheckoutEnabled() {
        return flags.get().isNewCheckoutEnabled();
    }

    public boolean isDarkModeEnabled() {
        return flags.get().isDarkModeEnabled();
    }

    public boolean isBetaUser(String userId) {
        return flags.get().getBetaUserIds().contains(userId);
    }
}
```

#### 예제 3: 다중 설정 파일 감시

```java
@Service
public class MultiConfigWatcherService {

    private final CentralDogma centralDogma;
    private final List<Watcher<?>> watchers = new ArrayList<>();

    @PostConstruct
    public void init() throws Exception {
        CentralDogmaRepository repo = centralDogma.forRepo("my-project", "config");

        // 여러 설정 파일을 동시에 Watch
        Watcher<JsonNode> dbWatcher = repo
            .watcher(Query.ofJsonPath("/database.json"))
            .start();

        Watcher<JsonNode> cacheWatcher = repo
            .watcher(Query.ofJsonPath("/cache.json"))
            .start();

        Watcher<JsonNode> apiWatcher = repo
            .watcher(Query.ofJsonPath("/api-keys.json"))
            .start();

        // 각각 리스너 등록
        dbWatcher.watch((rev, config) -> updateDatabaseConfig(config));
        cacheWatcher.watch((rev, config) -> updateCacheConfig(config));
        apiWatcher.watch((rev, config) -> updateApiKeys(config));

        watchers.addAll(List.of(dbWatcher, cacheWatcher, apiWatcher));

        // 모든 초기값 대기
        for (Watcher<?> watcher : watchers) {
            watcher.awaitInitialValue(10, TimeUnit.SECONDS);
        }
    }

    @PreDestroy
    public void destroy() {
        watchers.forEach(Watcher::close);
    }
}
```

#### 예제 4: 경로 패턴으로 여러 파일 감시

```java
@Service
public class DirectoryWatcherService {

    @PostConstruct
    public void init() throws Exception {
        CentralDogmaRepository repo = centralDogma.forRepo("my-project", "config");

        // /services/** 하위의 모든 JSON 파일 감시
        Watcher<Revision> watcher = repo
            .watcher(PathPattern.of("/services/**/*.json"))
            .start();

        watcher.watch((revision, rev) -> {
            log.info("Some config under /services changed at revision {}", revision);
            // 변경된 파일들 조회
            repo.file(PathPattern.of("/services/**/*.json"))
                .get(revision)
                .thenAccept(files -> {
                    files.forEach((path, entry) -> {
                        log.info("File: {}, Content: {}", path, entry.contentAsText());
                    });
                });
        });
    }
}
```

### 10.4 설정 변경 흐름 (Spring Boot)

```
┌─────────────────────────────────────────────────────────────────┐
│                      Spring Boot Application                     │
│                                                                 │
│  @PostConstruct                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  1. CentralDogma 클라이언트 (자동 주입)                    │   │
│  │  2. Watcher 생성 및 시작                                  │   │
│  │  3. 변경 리스너 등록                                      │   │
│  │  4. awaitInitialValue() - 초기값 대기                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Runtime: 설정 변경 감지                                   │   │
│  │  ┌───────────────────────────────────────────────────┐   │   │
│  │  │ watcher.watch((revision, config) -> {             │   │   │
│  │  │     // 1. 새 설정값 파싱                            │   │   │
│  │  │     // 2. AtomicReference/volatile 업데이트        │   │   │
│  │  │     // 3. 필요시 Bean 재구성                        │   │   │
│  │  │ });                                               │   │   │
│  │  └───────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  서비스 코드: 항상 최신 설정 사용                          │   │
│  │  ┌───────────────────────────────────────────────────┐   │   │
│  │  │ public void process() {                           │   │   │
│  │  │     DatabaseConfig config = configService.get();  │   │   │
│  │  │     // 항상 최신 설정으로 동작                       │   │   │
│  │  │ }                                                 │   │   │
│  │  └───────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 10.5 주의사항

1. **Thread Safety**: 설정 객체는 `volatile` 또는 `AtomicReference`로 관리
2. **초기화 대기**: `awaitInitialValue()`로 서비스 시작 전 초기 설정 로드 보장
3. **리소스 정리**: `@PreDestroy`에서 `watcher.close()` 필수
4. **에러 처리**: Watch 콜백에서 예외 발생 시 다음 Watch에 영향 없음
5. **백오프 설정**: 네트워크 장애 시 지수 백오프 자동 적용

### 10.6 런타임 Bean 재구성 전략

Spring Bean은 기본적으로 싱글톤으로 시작 시점에 생성되므로, 런타임에 "Bean 자체를 재구성"하는 것은 일반적이지 않습니다. 실제로는 다음 패턴들을 사용합니다.

#### 패턴 1: Bean은 그대로, 설정값만 동적으로 읽기 (권장)

가장 일반적인 접근법으로, Bean 자체는 유지하고 매번 최신 설정을 조회합니다:

```java
@Service
public class MyService {

    private final DynamicConfigService configService;

    public void process() {
        // Bean은 그대로, 매번 최신 설정을 조회
        DatabaseConfig config = configService.getDatabaseConfig();
        // config 값으로 로직 수행
    }
}
```

#### 패턴 2: Connection Pool 재구성 (새 인스턴스 교체)

HikariCP 같은 DataSource는 설정이 내부에 고정되므로, 새 인스턴스를 생성하고 교체해야 합니다:

```java
@Service
@Slf4j
public class DynamicDataSourceManager {

    private final AtomicReference<HikariDataSource> dataSourceRef = new AtomicReference<>();

    @PostConstruct
    public void init() {
        dataSourceRef.set(createDataSource(initialConfig));
    }

    public void onConfigChanged(DatabaseConfig newConfig) {
        HikariDataSource oldDs = dataSourceRef.get();
        HikariDataSource newDs = createDataSource(newConfig);

        // 새 DataSource로 교체
        dataSourceRef.set(newDs);

        // 기존 연결 풀 graceful shutdown (진행 중인 쿼리 완료 대기)
        if (oldDs != null) {
            CompletableFuture.runAsync(() -> {
                try {
                    Thread.sleep(30000); // 기존 트랜잭션 완료 대기
                    oldDs.close();
                    log.info("Old datasource closed");
                } catch (Exception e) {
                    log.error("Error closing old datasource", e);
                }
            });
        }
    }

    private HikariDataSource createDataSource(DatabaseConfig config) {
        HikariConfig hikariConfig = new HikariConfig();
        hikariConfig.setJdbcUrl("jdbc:mysql://" + config.getHost() + ":" + config.getPort() + "/db");
        hikariConfig.setUsername(config.getUsername());
        hikariConfig.setPassword(config.getPassword());
        hikariConfig.setMaximumPoolSize(config.getMaxPoolSize());
        return new HikariDataSource(hikariConfig);
    }

    public DataSource getDataSource() {
        return dataSourceRef.get();
    }
}
```

#### 패턴 3: Routing DataSource (더 안전한 방식)

Spring의 `AbstractRoutingDataSource`를 활용하여 무중단 전환:

```java
@Component
public class DynamicRoutingDataSource extends AbstractRoutingDataSource {

    private final AtomicReference<Object> currentKey = new AtomicReference<>("primary");
    private final Map<Object, DataSource> dataSources = new ConcurrentHashMap<>();

    @Override
    protected Object determineCurrentLookupKey() {
        return currentKey.get();
    }

    public void addDataSource(String key, DataSource dataSource) {
        dataSources.put(key, dataSource);
        setTargetDataSources(new HashMap<>(dataSources));
        afterPropertiesSet(); // 변경사항 적용
    }

    public void switchTo(String key) {
        if (dataSources.containsKey(key)) {
            String oldKey = (String) currentKey.getAndSet(key);
            log.info("Switched datasource from {} to {}", oldKey, key);

            // 이전 DataSource 정리 (비동기)
            scheduleCleanup(oldKey);
        }
    }
}
```

#### 패턴 4: HTTP Client 재구성

```java
@Service
@Slf4j
public class ConfigurableHttpClient {

    private final DynamicConfigService configService;
    private final AtomicReference<WebClient> clientRef = new AtomicReference<>();

    @PostConstruct
    public void init() {
        rebuildClient(configService.getApiConfig());

        // 설정 변경 감지
        configService.watchApiConfig(this::rebuildClient);
    }

    private void rebuildClient(ApiConfig config) {
        WebClient newClient = WebClient.builder()
            .baseUrl(config.getBaseUrl())
            .defaultHeader("Authorization", "Bearer " + config.getApiKey())
            .build();

        clientRef.set(newClient);
        log.info("WebClient rebuilt with new config");
    }

    public WebClient getClient() {
        return clientRef.get();
    }
}
```

#### 설정 종류별 재구성 전략

| 설정 종류 | 재구성 필요? | 권장 방법 |
|----------|------------|----------|
| **Feature Flag** | X | `AtomicReference`로 값만 교체 |
| **API Key** | X | `volatile` 변수로 관리 |
| **타임아웃 값** | X | 매 요청마다 최신값 조회 |
| **DB Connection Pool** | O | 새 Pool 생성 후 교체 |
| **Redis Connection** | O | Lettuce/Jedis 클라이언트 재생성 |
| **HTTP Client** | △ | 대부분 설정값만 교체 가능 |

#### 핵심 원칙

1. **Bean 자체를 재구성하는 것은 피한다** - 대신 설정값을 동적으로 읽음
2. **상태를 가진 리소스(Connection Pool 등)는 새 인스턴스 생성 후 교체**
3. **`AtomicReference`로 thread-safe하게 교체**
4. **기존 리소스는 graceful하게 정리** (진행 중인 작업 완료 후)

### 10.7 의존성 주입과 동적 설정 조회

이미 주입된 의존성에서 **설정값을 필드에 저장하지 않고, 사용 시점에 매번 조회**하는 것이 핵심입니다.

#### 잘못된 방식: 설정값을 필드에 저장

```java
@Service
public class PaymentService {
    private final int timeout;  // 주입 시점에 고정됨
    
    public PaymentService(AppConfig config) {
        this.timeout = config.getTimeout();  // 한 번만 읽음
    }
    
    public void pay() {
        // timeout은 애플리케이션 시작 시점 값으로 고정
        httpClient.setTimeout(timeout);
    }
}
```

이 방식의 문제점:
- `timeout` 값이 생성자 호출 시점에 고정됨
- Central Dogma에서 설정을 변경해도 반영되지 않음
- 설정 변경을 위해 애플리케이션 재시작 필요

#### 올바른 방식: 사용 시점에 조회

```java
@Service
public class PaymentService {
    private final Watcher<AppConfig> configWatcher;  // Watcher 주입
    
    public PaymentService(Watcher<AppConfig> configWatcher) {
        this.configWatcher = configWatcher;
    }
    
    public void pay() {
        // 매번 최신 설정값 조회
        AppConfig config = configWatcher.latest().value();
        httpClient.setTimeout(config.getTimeout());
    }
}
```

이 방식의 장점:
- 설정 변경 즉시 반영
- 애플리케이션 재시작 불필요
- Thread-safe (Watcher 내부에서 보장)

#### 성능 우려?

`Watcher.latest()`는 **내부적으로 캐싱된 값을 반환**하므로 성능 걱정이 없습니다:

```
┌─────────────────────────────────────────────────────────────┐
│                    Watcher 내부 구조                         │
│                                                             │
│  ┌─────────────────┐      ┌─────────────────────────────┐  │
│  │   latest()      │─────►│  메모리 캐시 (즉시 반환)      │  │
│  │   O(1) 조회     │      │  - volatile 변수             │  │
│  └─────────────────┘      │  - 네트워크 호출 없음         │  │
│                           └─────────────────────────────┘  │
│                                      ▲                      │
│                                      │ 설정 변경 시에만      │
│  ┌─────────────────┐                 │ 서버에서 업데이트     │
│  │  Long Polling   │─────────────────┘                      │
│  │  (백그라운드)    │                                        │
│  └─────────────────┘                                        │
└─────────────────────────────────────────────────────────────┘
```

- **매 호출마다 네트워크 요청이 발생하지 않음**
- 설정이 변경될 때만 서버에서 새 값을 받아옴
- `latest()`는 단순히 메모리에 캐싱된 값을 반환 (O(1))

#### 비교 정리

| 구분 | 필드 저장 방식 | 동적 조회 방식 |
|------|---------------|---------------|
| 설정 변경 반영 | 재시작 필요 | 즉시 반영 |
| 성능 | 약간 빠름 | Watcher 내부 캐싱으로 거의 동일 |
| 코드 복잡도 | 단순 | 약간 증가 |
| 권장 여부 | 정적 설정에만 | **동적 설정에 권장** |

## 참고 코드 위치

| 구성요소 | 파일 경로 | 주요 메서드 |
|---------|----------|------------|
| Watch 인터페이스 | `client/java/.../Watcher.java` | `latest()`, `watch()` |
| Watch 구현체 | `client/java/.../AbstractWatcher.java` | `scheduleWatch()`, `doWatch()` |
| Watch 요청 변환 | `server/.../WatchRequestConverter.java` | `convertRequest()`, `extractRevision()` |
| Watch 서비스 | `server/.../WatchService.java` | `watchFile()`, `scheduleTimeout()` |
| Git 리포지토리 | `server/.../GitRepository.java` | `watch()`, `notifyWatchers()` |
| Watch 관리자 | `server/.../CommitWatchers.java` | `add()`, `notify()` |
| 커밋 실행기 | `server/.../CommitExecutor.java` | `execute()`, `commit()` |
| 리비전 DB | `server/.../DefaultCommitIdDatabase.java` | `get()`, `put()` |
| 경로 패턴 | `server/.../PathPatternFilter.java` | `of()`, `matches()` |
| Spring Boot 설정 | `client/java-spring-boot3-autoconfigure/.../CentralDogmaSettings.java` | `getHosts()`, `getAccessToken()` |

---

**GitHub Repository**: https://github.com/line/centraldogma
