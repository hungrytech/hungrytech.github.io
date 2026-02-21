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

---

**GitHub Repository**: https://github.com/line/centraldogma