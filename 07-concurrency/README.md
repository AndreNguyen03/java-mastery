# Phase 07 — Concurrency

> **Mục tiêu:** Hiểu tại sao concurrent code khó, nhận dạng race condition / deadlock / starvation, và biết dùng đúng primitives của JDK.

---

## Part 1 — Thread Basics

### Process vs Thread
```
Process = độc lập, memory riêng
Thread  = chia sẻ memory với thread cùng process
```

### Thread lifecycle
```
NEW → RUNNABLE → (RUNNING) → BLOCKED / WAITING / TIMED_WAITING → TERMINATED
```

### 3 vấn đề core của Concurrency

**1. Visibility** — Thread A write, Thread B có thể không thấy giá trị mới (CPU cache, reordering).

**2. Atomicity** — `i++` là 3 operations: read, increment, write → không atomic.

**3. Ordering** — JVM/CPU có thể reorder instructions miễn là single-threaded behavior giữ nguyên.

---

## Part 2 — Synchronization Primitives

### `synchronized`
```java
// Synchronized method — lock trên this (instance)
public synchronized void increment() { count++; }

// Synchronized block — explicit lock object (prefer này)
private final Object lock = new Object();
public void increment() {
    synchronized (lock) { count++; }
}
```
**Monitor lock:** Mỗi Object có một intrinsic lock. `synchronized` acquire trước khi vào, release khi ra.

### `volatile`
```java
private volatile boolean running = true;
```
- Đảm bảo **visibility** — write visible ngay cho tất cả threads
- **Không đảm bảo atomicity** — `volatile int i; i++` vẫn race condition
- Dùng cho: flags, trạng thái đơn giản không có read-modify-write

### `java.util.concurrent.atomic`
```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();          // atomic read-modify-write
counter.compareAndSet(expect, update); // CAS
```
**CAS (Compare-And-Swap):** Hardware instruction, không cần lock, non-blocking.

### `ReentrantLock`
```java
Lock lock = new ReentrantLock();
lock.lock();
try {
    // critical section
} finally {
    lock.unlock(); // LUÔN unlock trong finally
}

// Với timeout
if (lock.tryLock(5, TimeUnit.SECONDS)) { ... }
```
**Khi nào dùng thay synchronized:**
- Cần `tryLock()` (non-blocking attempt)
- Cần fairness (`new ReentrantLock(true)`)
- Cần interrupt waiting thread

### `ReadWriteLock`
```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();
// Nhiều reader đồng thời, chỉ 1 writer tại một thời điểm
rwLock.readLock().lock();
rwLock.writeLock().lock();
```

---

## Part 3 — Executor Framework

### Thread Pool
```
Task → BlockingQueue → ThreadPoolExecutor → Worker Thread
                                              Worker Thread
                                              Worker Thread
```

```java
// Factory methods
ExecutorService pool = Executors.newFixedThreadPool(4);
ExecutorService pool = Executors.newCachedThreadPool();
ExecutorService pool = Executors.newSingleThreadExecutor();
ScheduledExecutorService sched = Executors.newScheduledThreadPool(2);

// Tạo thủ công để control fully
ExecutorService pool = new ThreadPoolExecutor(
    4,                              // corePoolSize
    8,                              // maximumPoolSize
    60L, TimeUnit.SECONDS,          // keepAliveTime
    new LinkedBlockingQueue<>(1000), // workQueue
    new ThreadPoolExecutor.CallerRunsPolicy() // rejection policy
);
```

### Future & Callable
```java
Future<String> future = executor.submit(() -> expensiveComputation());
// ... làm việc khác
String result = future.get(5, TimeUnit.SECONDS); // blocking với timeout
```

### CompletableFuture
```java
CompletableFuture
    .supplyAsync(() -> fetchUser(id))           // async
    .thenApply(user -> enrichWithProfile(user)) // transform (non-blocking)
    .thenCompose(user -> fetchOrders(user.id())) // chain async
    .thenAccept(orders -> sendResponse(orders))  // consume
    .exceptionally(ex -> handleError(ex));       // error handling

// Combine
CompletableFuture<User> userFuture = fetchUser(id);
CompletableFuture<Profile> profileFuture = fetchProfile(id);
CompletableFuture.allOf(userFuture, profileFuture)
    .thenRun(() -> buildResponse(userFuture.join(), profileFuture.join()));
```

---

## Part 4 — Concurrent Collections

| Collection | Thread-safe | Notes |
|-----------|------------|-------|
| `ConcurrentHashMap` | Yes | Segment locking (Java 8: CAS + synchronized per bucket) |
| `CopyOnWriteArrayList` | Yes | Copy on write — read fast, write slow, good for rare writes |
| `BlockingQueue` | Yes | `put()` blocks khi đầy, `take()` blocks khi trống — Producer-Consumer |
| `ConcurrentLinkedQueue` | Yes | Non-blocking, CAS-based |

---

## Part 5 — Coordination

```java
// CountDownLatch — wait for N events
CountDownLatch latch = new CountDownLatch(3);
// 3 worker threads gọi latch.countDown() khi xong
latch.await(); // main thread chờ cho đến khi count = 0

// CyclicBarrier — tất cả threads sync tại một điểm
CyclicBarrier barrier = new CyclicBarrier(4);
// Mỗi thread gọi barrier.await() — tất cả chờ cho đến khi đủ 4

// Semaphore — giới hạn concurrent access
Semaphore semaphore = new Semaphore(10); // max 10 concurrent
semaphore.acquire();
try { accessResource(); }
finally { semaphore.release(); }
```

---

## Part 6 — Deadlock, Starvation, Livelock

### Deadlock
```
Thread A holds Lock 1, waits for Lock 2
Thread B holds Lock 2, waits for Lock 1
→ cả hai chờ mãi mãi
```
**Phòng tránh:** Acquire locks theo thứ tự cố định.

**Detect:** `jstack <pid>` sẽ hiện "DEADLOCK FOUND".

### Starvation
Thread không bao giờ được CPU vì thread priority cao liên tục chiếm.

### Livelock
Threads active nhưng không tiến triển — liên tục phản ứng với nhau.

---

## Part 7 — Java Memory Model & Happens-Before

**Happens-before guarantee:** Nếu A happens-before B, thì tất cả write trong A visible cho B.

Rules:
- `synchronized` block unlock happens-before subsequent lock
- `volatile` write happens-before subsequent read
- Thread start happens-before first action in thread
- `Thread.join()` returns after thread terminates

---

## Part 8 — Virtual Threads (Java 21)

```java
// Platform thread: 1 Java thread = 1 OS thread (stack ~1MB)
// Virtual thread: lightweight, managed by JVM, millions possible

Thread.ofVirtual().start(() -> handleRequest());
ExecutorService vexec = Executors.newVirtualThreadPerTaskExecutor();
```

**Khi nào Virtual Threads tốt:** I/O-bound tasks (HTTP, DB, file) — thread mount/unmount tự động khi blocking.

**Khi nào không tốt:** CPU-bound tasks — virtual thread không giải quyết CPU contention.

---

## Project — Concurrent Task Scheduler

**Không viết ThreadPoolExecutor từ đầu. Dùng JDK và tạo scenarios.**

```
src/
├── scheduler/
│   ├── TaskScheduler.java          ← wrapper around ScheduledExecutorService
│   ├── PriorityTaskQueue.java      ← PriorityBlockingQueue
│   └── TaskResult.java             (Record)
├── scenarios/
│   ├── RaceConditionDemo.java      ← shared counter, multiple threads
│   ├── DeadlockDemo.java           ← hai lock, thứ tự ngược nhau
│   ├── ProducerConsumerDemo.java   ← BlockingQueue
│   ├── ThreadPoolExhaustionDemo.java ← queue full, rejection policy
│   └── VirtualThreadDemo.java      ← so sánh platform vs virtual threads
├── benchmark/
│   └── ThreadPoolBenchmark.java    ← fixed vs cached vs virtual
└── test/
    ├── TaskSchedulerTest.java
    └── ConcurrentCounterTest.java  ← RepeatedTest để catch race
```

**Yêu cầu:**
1. Reproduce race condition → fix bằng `AtomicInteger` hoặc `synchronized`
2. Reproduce deadlock → fix bằng lock ordering
3. Producer-Consumer với `BlockingQueue` — đúng backpressure
4. So sánh throughput: fixed thread pool vs virtual threads cho I/O-bound task

---

## Checklist — 6 câu hỏi

Ví dụ với `CompletableFuture`:

| # | Câu hỏi | Trả lời |
|---|---------|---------|
| 1 | Là gì? | Non-blocking async computation — future with callback chaining |
| 2 | Giải quyết gì? | Tránh callback hell, compose async operations một cách readable |
| 3 | Hoạt động thế nào? | Wrap async task, chain transformations, run on ForkJoinPool by default |
| 4 | Khi nào dùng? | Multiple independent async calls, orchestrate parallel fetches |
| 5 | Khi nào không? | Simple sequential async — Future đủ; reactive stream cần Reactor/RxJava |
| 6 | Debug thế nào? | `exceptionally()` để catch; thread dump để xem stuck; timeout với `orTimeout()` |

---

## Run

```bash
mvn test -pl phase-07-concurrency
mvn compile exec:java -pl phase-07-concurrency \
    -Dexec.mainClass="com.nguyenngoc.phase07.scenarios.DeadlockDemo"
```
