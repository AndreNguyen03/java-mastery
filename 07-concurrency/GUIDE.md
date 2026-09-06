# Phase 07 — Concurrency · Lý thuyết & Lab

---

## 1. Lý thuyết

### 1.0 Thread Lifecycle & Process vs Thread

**Process vs Thread:**
| | Process | Thread |
|---|---|---|
| Memory | Riêng (isolated address space) | Shared heap trong cùng process |
| Communication | IPC (pipe, socket, shared mem) | Shared objects trực tiếp |
| Creation cost | Nặng (fork, new address space) | Nhẹ hơn (stack mới, registers) |
| Crash isolation | Process crash không ảnh hưởng khác | Thread crash có thể kill cả process |
| Context switch | Đắt (flush TLB, address space) | Rẻ hơn |

**Thread Lifecycle States:**
```
                    start()
NEW ──────────────────────────→ RUNNABLE
                                    │ ↑
               synchronized block  │ │ lock acquired
               or lock not avail.  ↓ │
                               BLOCKED
                                    
                    Object.wait()   │ ↑
                    or join()       ↓ │  notify()/notifyAll()/join completes
                               WAITING

                    wait(timeout)   │ ↑  
                    sleep(ms)       ↓ │  timeout/notify
                          TIMED_WAITING

                    run() completes
RUNNABLE ─────────────────────────→ TERMINATED
```

**Trạng thái chi tiết:**
- **NEW:** Thread được tạo nhưng chưa `start()`
- **RUNNABLE:** Đang chạy hoặc sẵn sàng chạy (scheduler quyết định khi nào CPU cho)
- **BLOCKED:** Đang chờ monitor lock (ai đó đang giữ `synchronized` block)
- **WAITING:** Đang chờ vô thời hạn: `Object.wait()`, `Thread.join()`, `LockSupport.park()`
- **TIMED_WAITING:** Chờ có timeout: `Thread.sleep(ms)`, `wait(ms)`, `join(ms)`
- **TERMINATED:** `run()` kết thúc (normal hoặc exception)

```java
Thread t = new Thread(() -> {
    System.out.println("Running: " + Thread.currentThread().getState()); // RUNNABLE
});
System.out.println("Before start: " + t.getState()); // NEW
t.start();
t.join();
System.out.println("After join: " + t.getState()); // TERMINATED
```

---

### 1.00 ReentrantLock — Flexible Locking

**Tại sao cần ReentrantLock khi đã có synchronized:**
- `synchronized` không thể tryLock (với timeout)
- `synchronized` không thể bị interrupted
- `synchronized` không có fairness (fair queuing)
- Không thể lock trong một method và unlock ở method khác

```java
private final ReentrantLock lock = new ReentrantLock(/* fair= */ false);
private final ReentrantLock fairLock = new ReentrantLock(true); // FIFO order

// Basic usage — must unlock in finally
lock.lock();
try {
    // critical section
} finally {
    lock.unlock(); // ALWAYS in finally — avoid deadlock on exception
}

// tryLock — avoid blocking indefinitely
if (lock.tryLock(500, TimeUnit.MILLISECONDS)) {
    try {
        // got the lock within 500ms
    } finally {
        lock.unlock();
    }
} else {
    // couldn't get lock — handle gracefully (retry, fallback, throw)
    log.warn("Could not acquire lock — skipping");
}

// lockInterruptibly — allow thread to be interrupted while waiting
try {
    lock.lockInterruptibly(); // throws InterruptedException if thread.interrupt() called
    try {
        // critical section
    } finally {
        lock.unlock();
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // restore interrupt flag
    log.info("Thread interrupted while waiting for lock");
}
```

**Reentrant nghĩa là gì:** Cùng thread có thể acquire lock nhiều lần mà không deadlock. Lock count được tăng lên, và phải unlock tương ứng. `synchronized` cũng reentrant.

---

### 1.000 Synchronization Utilities

**CountDownLatch — "chờ N sự kiện xảy ra":**
```java
// Use case: main thread chờ 3 services khởi động xong mới proceed
CountDownLatch latch = new CountDownLatch(3); // count = 3

// Service threads đếm ngược:
ExecutorService executor = Executors.newFixedThreadPool(3);
executor.submit(() -> { initUserService();    latch.countDown(); }); // count = 2
executor.submit(() -> { initOrderService();   latch.countDown(); }); // count = 1
executor.submit(() -> { initPaymentService(); latch.countDown(); }); // count = 0

// Main thread chờ:
latch.await(); // blocks until count = 0
System.out.println("All services ready — starting server");

// Với timeout:
boolean allReady = latch.await(30, TimeUnit.SECONDS);
if (!allReady) throw new TimeoutException("Services failed to start within 30s");

// ⚠️ CountDownLatch không reusable — một lần xong là xong
```

**CyclicBarrier — "tất cả threads chờ nhau ở một điểm":**
```java
// Use case: parallel computation — tất cả workers phải hoàn thành phase 1 trước khi phase 2
CyclicBarrier barrier = new CyclicBarrier(
    3, // number of threads
    () -> System.out.println("All workers reached barrier — starting next phase") // optional action
);

// Worker threads:
Runnable worker = () -> {
    processPhase1();
    try {
        barrier.await(); // wait for all 3 workers to reach here
    } catch (BrokenBarrierException e) { /* handle */ }
    processPhase2(); // only runs after all workers finish phase 1
};

// CyclicBarrier reusable — sau khi barrier breaks, resets for next use
// Phân biệt vs CountDownLatch:
// - CountDownLatch: 1 thread chờ N events (one-directional)
// - CyclicBarrier: N threads chờ nhau (mutual wait), reusable
```

**Semaphore — "giới hạn concurrency":**
```java
// Use case: rate limiting, connection pool, limit concurrent DB connections
Semaphore semaphore = new Semaphore(5); // max 5 concurrent threads

// HTTP client limiting concurrent requests:
public Response fetch(String url) throws InterruptedException {
    semaphore.acquire(); // blocks if 5 permits already taken
    try {
        return httpClient.get(url);
    } finally {
        semaphore.release(); // ALWAYS release in finally
    }
}

// Tryacquire with timeout:
if (semaphore.tryAcquire(1, TimeUnit.SECONDS)) {
    try { /* ... */ } finally { semaphore.release(); }
} else {
    throw new TooManyRequestsException("Rate limit exceeded");
}
```

**Starvation vs Livelock:**

```java
// STARVATION: một thread không bao giờ được CPU/resource
// Nguyên nhân: 
// - Thread priority thấp → scheduler luôn chọn thread ưu tiên cao
// - Unfair lock — cùng thread giành lock mãi
// Fix: Fair lock (ReentrantLock(true)), tránh quá chênh lệch priority

// LIVELOCK: threads phản ứng với nhau → không ai tiến được, nhưng không block
// Ví dụ:
// Thread A: "bạn đi trước" → nhường cho B
// Thread B: "không, bạn đi trước" → nhường lại cho A
// Cả hai cứ nhường nhau — active nhưng không tiến

// Ví dụ code:
class LobbySimulation {
    volatile boolean aMovedRight = false;
    volatile boolean bMovedRight = false;
    
    void personA() {
        while (true) {
            if (bMovedRight) { aMovedRight = true; return; } // A ngồi xuống
            bMovedRight = false; // "bạn ngồi xuống đi"
        }
    }
    // → livelock nếu không ai chịu ngồi xuống trước
    
    // Fix: randomized backoff, designated "leader" (lower ID goes first)
}
```

---

### 1.1 Race Condition — Tại sao xảy ra

**Race condition:** Kết quả phụ thuộc vào timing của threads — kết quả không deterministic.

```java
// Tưởng chừng đơn giản nhưng thread-unsafe:
private int counter = 0;
counter++; // KHÔNG phải atomic!

// Thực ra là 3 bước:
// 1. READ  counter từ memory vào register
// 2. ADD   1 vào register
// 3. WRITE register về memory

// Thread 1: READ(0) → ADD → [context switch] → WRITE(1)
// Thread 2:            READ(0) → ADD → WRITE(1)
// Kết quả: 1 thay vì 2 → mất một increment!
```

**Fixes theo thứ tự preference:**

1. **`AtomicInteger.incrementAndGet()`** — CAS (Compare-And-Swap) ở CPU level, lock-free, nhanh nhất
2. **`synchronized`** — mutual exclusion, chậm hơn do OS lock, nhưng flexible hơn cho compound operations
3. **`volatile`** — chỉ đảm bảo visibility (không đảm bảo atomicity) — cho simple reads/writes của single variable

---

### 1.2 Java Memory Model (JMM)

**Visibility problem:** Mỗi CPU core có L1/L2 cache riêng. Thread A write biến X → lưu vào cache của core A → Thread B (trên core B) đọc X từ cache của mình → thấy giá trị cũ.

**`volatile` giải quyết visibility:**
- Write `volatile` var → flush tới main memory ngay lập tức
- Read `volatile` var → đọc từ main memory, không dùng cache
- Không đảm bảo atomicity: `volatile int x; x++` vẫn có race condition

**`synchronized` giải quyết cả visibility và atomicity:**
- Entering `synchronized` block: read fresh values từ main memory
- Exiting: flush writes tới main memory
- Mutual exclusion: chỉ 1 thread tại một thời điểm

**happens-before relationship:**
- Nếu action A happens-before B → mọi thứ A thấy, B cũng thấy
- synchronized unlock happens-before synchronized lock
- volatile write happens-before volatile read
- Thread.start() happens-before mọi action trong thread đó

---

### 1.3 CompletableFuture — Async Programming

**Problem với Future:**
`Future.get()` block thread hiện tại. Không compose được: "khi task A xong, chạy task B, khi cả A và B xong, chạy C".

**CompletableFuture giải quyết:**
- Non-blocking chains với callbacks
- Exception handling trong chain
- Combine nhiều futures

```
thenApply()     — transform kết quả (Function<T,U>)
thenAccept()    — consume kết quả (Consumer<T>)
thenCompose()   — chain future ra future (flatMap equivalent)
thenCombine()   — wait 2 futures, combine kết quả
allOf()         — wait tất cả futures complete
anyOf()         — complete khi bất kỳ future nào complete
exceptionally() — handle exception, return fallback
```

**Thread pool:**
- Mặc định: `ForkJoinPool.commonPool()` — shared, bounded by CPU cores
- IO-bound tasks: nên dùng executor riêng với unbounded thread count vì IO block thread

---

### 1.4 BlockingQueue — Producer-Consumer Pattern

**Tại sao BlockingQueue:**
Producer tạo data nhanh hơn consumer xử lý → không có buffer → producer bị throttle hoặc data bị drop.

```
Producer → [BlockingQueue] → Consumer
           (bounded buffer)
```

- `put()`: block nếu queue full (producer throttled — backpressure)
- `take()`: block nếu queue empty (consumer waits)
- `poll(timeout, unit)`: wait với timeout, trả null nếu timeout — tốt cho graceful shutdown

**Bounded vs Unbounded:**
- `LinkedBlockingQueue(100)`: bounded — backpressure nếu consumer chậm
- `LinkedBlockingQueue()`: unbounded — có thể OutOfMemory nếu consumer quá chậm

---

### 1.5 ThreadPool — ExecutorService

**Tại sao không `new Thread()` trực tiếp:**
1. Thread creation overhead (~1ms, ~1MB stack)
2. Không bounded — unlimited threads → OutOfMemory
3. Không lifecycle management

**ThreadPoolExecutor parameters:**
```java
new ThreadPoolExecutor(
    corePoolSize,     // luôn alive, kể cả idle
    maximumPoolSize,  // max khi queue full
    keepAliveTime,    // idle non-core thread alive time
    timeUnit,
    new LinkedBlockingQueue<>(100), // task queue
    threadFactory,
    rejectionHandler  // khi queue full VÀ maxPoolSize reached
)
```

**RejectionPolicies:**
- `AbortPolicy` (default): throw RejectedExecutionException
- `CallerRunsPolicy`: caller thread executes task — natural backpressure
- `DiscardPolicy`: silently drop task
- `DiscardOldestPolicy`: drop oldest queued task, retry new one

---

### 1.6 Virtual Threads (Java 21) — Game Changer

**Problem với platform threads (OS threads):**
- Mỗi Java thread = 1 OS thread = ~1MB stack = limited by OS
- Typical app: 200 threads max (với 200MB RAM cho threads)
- IO-bound task: thread block → wasting OS thread while waiting

**Virtual Thread:**
- Mapped to carrier (OS) thread, bur many VTs share one carrier
- JVM unmounts VT from carrier khi VT blocks on IO → carrier thread free để chạy other VT
- Có thể tạo hàng triệu VTs

```
Platform Thread Model:     Virtual Thread Model:
┌──────┐ ┌──────┐          ┌──────┐ ┌──────┐
│Java  │ │Java  │          │  VT  │ │  VT  │ ← millions
│Thread│ │Thread│          │      │ │      │
└──┬───┘ └──┬───┘          └──┬───┘ └──┬───┘
   │         │                │(mounted) │
┌──┴─────────┴──┐         ┌───┴─────────┴──┐
│OS Thread 1    │         │OS Thread 1     │ ← just a few
│OS Thread 2    │         │OS Thread 2     │
└───────────────┘         └────────────────┘
```

**Khi nào dùng Virtual Threads:**
✅ IO-bound tasks (HTTP calls, DB queries, file IO)
❌ CPU-bound tasks (no benefit — VT still needs CPU)
❌ `synchronized` blocks with IO (VT pinned to carrier — giảm benefit)

**Thay thế `synchronized` bằng `ReentrantLock` trong virtual thread code:**
```java
// ❌ synchronized pins virtual thread to carrier
synchronized (lock) {
    Thread.sleep(1000); // VT pinned, carrier can't do other work
}

// ✅ ReentrantLock: JVM can unmount VT while waiting
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    Thread.sleep(1000); // VT can be unmounted if lock isn't held
} finally {
    lock.unlock();
}
```

---

## 2. Vấn đề thường gặp & Cách fix

### Issue 1: Deadlock

**Nguyên nhân:** Circular lock dependency. Thread A giữ lock 1, đợi lock 2. Thread B giữ lock 2, đợi lock 1.

```java
// ❌ Deadlock potential
void transfer(Account from, Account to, double amount) {
    synchronized (from) { // Thread A locks Account1
        synchronized (to) { // Thread B locks Account2
            // Nếu Thread B calls transfer(account2, account1) → deadlock
        }
    }
}

// ✅ Fix: consistent lock ordering (by ID)
void transfer(Account from, Account to, double amount) {
    Account first = from.getId() < to.getId() ? from : to;
    Account second = from.getId() < to.getId() ? to : from;
    synchronized (first) {
        synchronized (second) {
            // always acquire in same order → no cycle
        }
    }
}
```

### Issue 2: ThreadLocal leak trong Thread pools

```java
// ❌ ThreadLocal không được clear → thread được reuse trong pool giữ giá trị cũ
static ThreadLocal<UserContext> context = new ThreadLocal<>();

void handleRequest(User user) {
    context.set(new UserContext(user));
    doWork(); // works fine
    // Không clear → next request (same thread) sees previous user's context!
}

// ✅ Clear trong finally
void handleRequest(User user) {
    try {
        context.set(new UserContext(user));
        doWork();
    } finally {
        context.remove(); // PHẢI remove, không phải set null
    }
}
```

### Issue 3: CompletableFuture exception không được handled

```java
// ❌ Exception bị "nuốt" — không ai thấy lỗi này
CompletableFuture.runAsync(() -> {
    throw new RuntimeException("oops");
});
// → swallowed! logs không hiện gì

// ✅ Luôn handle exception
CompletableFuture.runAsync(() -> {
    throw new RuntimeException("oops");
}).exceptionally(ex -> {
    log.error("Async task failed", ex);
    return null;
});

// Hoặc khi collect kết quả
try {
    future.get(5, TimeUnit.SECONDS);
} catch (ExecutionException e) {
    log.error("Task failed", e.getCause()); // getCause() = original exception
}
```

---

## 3. Code mẫu

```java
// RaceConditionDemo.java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

public class RaceConditionDemo {

    public static void main(String[] args) throws InterruptedException {
        int threadCount = 100, incrementsPerThread = 1000;
        int expected = threadCount * incrementsPerThread;

        // ❌ Unsafe
        int[] unsafeCounter = {0};
        runConcurrently(threadCount, () -> {
            for (int i = 0; i < incrementsPerThread; i++) unsafeCounter[0]++;
        });
        System.out.printf("Unsafe:      expected=%d, actual=%d (%s)%n",
            expected, unsafeCounter[0], unsafeCounter[0] == expected ? "✅" : "❌ RACE CONDITION");

        // ✅ AtomicInteger
        AtomicInteger atomicCounter = new AtomicInteger(0);
        runConcurrently(threadCount, () -> {
            for (int i = 0; i < incrementsPerThread; i++) atomicCounter.incrementAndGet();
        });
        System.out.printf("Atomic:      expected=%d, actual=%d (%s)%n",
            expected, atomicCounter.get(), atomicCounter.get() == expected ? "✅" : "❌");

        // ✅ synchronized
        int[] syncCounter = {0};
        Object lock = new Object();
        runConcurrently(threadCount, () -> {
            for (int i = 0; i < incrementsPerThread; i++) {
                synchronized (lock) { syncCounter[0]++; }
            }
        });
        System.out.printf("Synchronized: expected=%d, actual=%d (%s)%n",
            expected, syncCounter[0], syncCounter[0] == expected ? "✅" : "❌");
    }

    static void runConcurrently(int threadCount, Runnable task) throws InterruptedException {
        CountDownLatch start = new CountDownLatch(1);
        CountDownLatch done = new CountDownLatch(threadCount);
        for (int i = 0; i < threadCount; i++) {
            new Thread(() -> {
                try { start.await(); task.run(); } catch (InterruptedException e) {}
                done.countDown();
            }).start();
        }
        start.countDown(); // release all threads simultaneously
        done.await();
    }
}
```

```java
// AsyncOrchestrator.java
import java.util.concurrent.*;

public class AsyncOrchestrator {
    private final ExecutorService executor;

    public AsyncOrchestrator(ExecutorService executor) {
        this.executor = executor;
    }

    /**
     * Parallel fetch: user + orders + inventory — wait for all
     * Nếu sequential: 3 × ~100ms = 300ms
     * Nếu parallel:   max(~100ms) = 100ms
     */
    public Dashboard buildDashboard(String userId) {
        CompletableFuture<UserProfile> userFuture =
            CompletableFuture.supplyAsync(() -> fetchUser(userId), executor);

        CompletableFuture<List<Order>> ordersFuture =
            CompletableFuture.supplyAsync(() -> fetchOrders(userId), executor);

        CompletableFuture<List<Product>> inventoryFuture =
            CompletableFuture.supplyAsync(() -> fetchInventory(userId), executor);

        // Wait for all three, combine
        return CompletableFuture.allOf(userFuture, ordersFuture, inventoryFuture)
            .thenApply(_ -> new Dashboard(
                userFuture.join(),      // join() doesn't throw checked exception
                ordersFuture.join(),
                inventoryFuture.join()
            ))
            .exceptionally(ex -> {
                log.error("Dashboard build failed for userId={}", userId, ex);
                return Dashboard.empty(); // fallback
            })
            .join();
    }

    /**
     * Sequential pipeline: step 2 depends on step 1 result
     */
    public CompletableFuture<OrderConfirmation> pipeline(CreateOrderRequest request) {
        return CompletableFuture
            .supplyAsync(() -> validateOrder(request), executor)    // step 1
            .thenCompose(order -> chargePayment(order))             // step 2 (returns future)
            .thenCompose(paymentResult -> reserveInventory(paymentResult.orderId())) // step 3
            .thenApply(reservation -> new OrderConfirmation(reservation.orderId()))
            .exceptionally(ex -> {
                // Handle failure, trigger compensation if needed
                log.error("Order pipeline failed", ex);
                throw new OrderProcessingException("Order failed", ex);
            });
    }
}
```

```java
// VirtualThreadDemo.java
import java.util.concurrent.*;

public class VirtualThreadDemo {

    static final int TASK_COUNT = 10_000;
    static final int TASK_DURATION_MS = 100; // simulates IO

    public static void main(String[] args) throws Exception {
        System.out.println("Running " + TASK_COUNT + " tasks each taking " + TASK_DURATION_MS + "ms");

        // Platform threads: limited by OS thread count
        long ptStart = System.currentTimeMillis();
        try (ExecutorService exec = Executors.newFixedThreadPool(200)) {
            runTasks(exec, TASK_COUNT);
        }
        System.out.printf("Platform threads (200): %d ms%n", System.currentTimeMillis() - ptStart);

        // Virtual threads: one per task — 10K concurrent VTs
        long vtStart = System.currentTimeMillis();
        try (ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor()) {
            runTasks(exec, TASK_COUNT);
        }
        System.out.printf("Virtual threads:        %d ms%n", System.currentTimeMillis() - vtStart);
        // Platform: ~5000ms (10000 tasks / 200 threads × 100ms)
        // Virtual:  ~100ms  (all run concurrently)
    }

    static void runTasks(ExecutorService exec, int count) throws Exception {
        CountDownLatch latch = new CountDownLatch(count);
        for (int i = 0; i < count; i++) {
            exec.submit(() -> {
                Thread.sleep(TASK_DURATION_MS); // simulates IO (DB query, HTTP call)
                latch.countDown();
                return null;
            });
        }
        latch.await(60, TimeUnit.SECONDS);
    }
}
```

```java
// ConcurrentCache.java — ReadWriteLock pattern
import java.util.concurrent.locks.*;

public class ConcurrentCache<K, V> {
    private final Map<K, V> store = new HashMap<>();
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private final Lock readLock = lock.readLock();
    private final Lock writeLock = lock.writeLock();

    /**
     * Multiple readers can hold readLock simultaneously.
     * readLock and writeLock are mutually exclusive.
     */
    public V get(K key) {
        readLock.lock();
        try {
            return store.get(key);
        } finally {
            readLock.unlock(); // PHẢI unlock trong finally
        }
    }

    public void put(K key, V value) {
        writeLock.lock();
        try {
            store.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }

    /**
     * Double-checked locking: avoid write lock if already cached.
     * Tránh unnecessary write lock acquisition khi cache hit.
     */
    public V getOrCompute(K key, java.util.function.Supplier<V> loader) {
        // First check with read lock (cheap — multiple readers allowed)
        readLock.lock();
        try {
            V existing = store.get(key);
            if (existing != null) return existing;
        } finally {
            readLock.unlock();
        }

        // Must compute — acquire write lock
        writeLock.lock();
        try {
            // Check again (another thread may have computed between our locks)
            V existing = store.get(key);
            if (existing != null) return existing;

            V value = loader.get();
            store.put(key, value);
            return value;
        } finally {
            writeLock.unlock();
        }
    }
}
```

---

## 4. Lab Steps

1. **Race condition:** Chạy `RaceConditionDemo.main()` nhiều lần → unsafe counter khác nhau mỗi lần
2. **Virtual threads:** So sánh thời gian 10K IO tasks với platform vs virtual threads
3. **Producer-consumer:** Implement `ProducerConsumer` với `LinkedBlockingQueue(10)`, producer nhanh hơn consumer — observe backpressure
4. **Deadlock:** Tạo 2 threads, simulate transfer theo cả hai hướng → `jstack` tìm deadlock → fix với lock ordering
5. **ReadWriteLock:** Benchmark cache reads: synchronized vs ReadWriteLock (reads trong parallel)

---

## 5. Checklist tự kiểm tra

- [ ] `counter++` không atomic — 3 steps: read, increment, write
- [ ] `volatile` đảm bảo visibility nhưng không atomicity
- [ ] `CompletableFuture.allOf()` không trả result — phải `.join()` từng future riêng
- [ ] Virtual threads tốt cho IO-bound, không giúp gì cho CPU-bound
- [ ] `synchronized` trong VT code → pinning → giảm hiệu quả — dùng `ReentrantLock`
- [ ] ThreadLocal: `.remove()` trong finally — không thể bỏ qua trong thread pool
- [ ] `ReadWriteLock`: nhiều readers có thể concurrent, reader và writer exclusive
