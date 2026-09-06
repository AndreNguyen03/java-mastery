# Phase 06 — JVM Memory & Performance · Lý thuyết & Lab

---

## 1. Lý thuyết

### 1.1 JVM Memory Architecture

**JVM chia bộ nhớ thành các vùng:**

```
┌─────────────────────────────────────────────────┐
│                     HEAP                         │
│  ┌──────────────────┐  ┌─────────────────────┐  │
│  │   Young Gen       │  │     Old Gen          │  │
│  │  ┌─────┬───────┐ │  │  (long-lived objects)│  │
│  │  │Eden │S0 │S1 │ │  │                      │  │
│  │  └─────┴───────┘ │  └─────────────────────┘  │
│  └──────────────────┘                            │
└─────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────┐
│              Metaspace (off-heap)                 │
│        Class metadata, static data               │
└─────────────────────────────────────────────────┘
┌──────────────┐  ┌─────────────┐  ┌────────────┐
│ Thread Stacks │  │  Code Cache │  │  Direct Mem │
└──────────────┘  └─────────────┘  └────────────┘
```

**Young Generation (Minor GC):**
- **Eden:** Tất cả objects mới được tạo ở đây
- **Survivor 0/1:** Objects sống qua Minor GC được copy qua đây. Sau N lần survive (default 15), promote lên Old Gen
- Minor GC rất nhanh (< 100ms) vì chỉ scan Young Gen — "generational hypothesis": hầu hết objects chết trẻ

**Old Generation (Major/Full GC):**
- Objects sống lâu — long-lived caches, static data, session objects
- Full GC scan toàn bộ Heap → chậm (có thể hàng giây) → "Stop the World"

**Metaspace (Java 8+, thay PermGen):**
- Lưu class metadata, bytecode, method signatures
- Không có hard limit mặc định (PermGen có) → có thể grow đến native memory
- ClassLoader leak → Metaspace leak → `OutOfMemoryError: Metaspace`

---

### 1.2 Garbage Collection Algorithms

**G1GC (Garbage First — default từ Java 9):**
- Chia heap thành regions bằng nhau (~2048 regions)
- Collect regions có nhiều garbage nhất trước (vì vậy gọi là "Garbage First")
- Concurrent phases: không cần stop-the-world hoàn toàn
- Cấu hình: `-XX:MaxGCPauseMillis=200` — target pause goal (không đảm bảo tuyệt đối)

**ZGC (Java 15+ production ready):**
- Concurrent collection: pause time < 1ms (pause chỉ để mark roots)
- Trade-off: higher CPU usage do concurrent work
- Thích hợp: latency-sensitive applications (trading, gaming)

**GC metrics cần theo dõi:**
- `GCCount`: Minor GC nhiều là bình thường, Full GC ít
- `GCTime`: % time spent in GC — > 5% là vấn đề
- `OldUsed`: nếu OldUsed tăng liên tục mà không giảm → memory leak

---

### 1.3 Memory Leaks — Nguyên nhân phổ biến

**1. Static collection không có eviction:**
```java
// ❌ Cache lớn dần mãi
class SessionCache {
    private static final Map<String, Session> cache = new HashMap<>(); // NEVER shrinks
    static void put(String id, Session s) { cache.put(id, s); }
}
```
Object held by static reference → GC roots → không bao giờ collected.

**2. ThreadLocal không được cleared:**
```java
// ❌ Thread pool reuse threads — ThreadLocal tồn tại mãi
static ThreadLocal<HeavyObject> local = new ThreadLocal<>();
void serve(Request req) {
    local.set(new HeavyObject());
    // process...
    // KHÔNG call local.remove() → HeavyObject sống mãi với thread
}
```

**3. Inner class giữ reference đến outer class:**
```java
// ❌ Anonymous Runnable giữ reference đến Activity (Android pattern)
button.setOnClickListener(new View.OnClickListener() {
    void onClick(View v) {
        // this$ reference đến enclosing class
        SomeLargeObject.this.doSomething(); // implicit reference
    }
});
// Nếu Runnable được submitted và outlives Activity → Activity không được GC
```

**4. Unclosed resources:**
```java
// ❌ Connection không được close → connection pool exhaustion
Connection conn = dataSource.getConnection();
// Nếu exception → conn không được closed → leak
ResultSet rs = conn.createStatement().executeQuery("...");

// ✅ Try-with-resources đảm bảo close() được gọi dù exception
try (Connection conn = dataSource.getConnection();
     Statement stmt = conn.createStatement();
     ResultSet rs = stmt.executeQuery("...")) {
    // process
}
```

---

### 1.4 JIT Compilation

**Interpretation → Compilation pipeline:**
1. JVM interpret bytecode lúc đầu
2. JIT profiling: đếm method call count
3. Khi method được gọi nhiều (hot) → C1 compiler (quick compile, basic optimizations)
4. Khi rất hot → C2 compiler (aggressive optimizations: inlining, loop unrolling, escape analysis)

**Escape analysis:** Nếu object không "escape" method (không return, không store vào field) → JIT có thể **stack-allocate** thay vì heap → không cần GC cho object đó.

**Warmup:** JVM cần thời gian để JIT-compile hot paths. Benchmark đầu tiên chậm vì interpret. Warmup = run benchmark nhiều lần trước khi measure.

---

### 1.5 Diagnostic Tools

**jstat** — GC statistics realtime:
```bash
jstat -gc <pid> 5000  # mỗi 5 giây
# Columns quan trọng:
# S0U/S1U: Survivor usage
# EU: Eden usage
# OU: Old usage
# MU: Metaspace usage
# GCT: Total GC time (seconds)
# YGCT: Young GC time
# FGCT: Full GC time
```

**jstack** — Thread dump (xem deadlock, blocked threads):
```bash
jstack <pid> > thread-dump.txt
# Tìm: "BLOCKED", "WAITING", "deadlock detected"
```

**jmap** — Heap dump và histogram:
```bash
jmap -histo <pid> | head -30  # top 30 classes by instance count
jmap -dump:format=b,file=heap.hprof <pid>  # full heap dump → analyze with Eclipse MAT
```

**jcmd** — Modern tool (preferred):
```bash
jcmd <pid> VM.flags          # JVM flags hiện tại
jcmd <pid> GC.heap_info      # heap regions info (G1)
jcmd <pid> Thread.print      # thread dump
jcmd <pid> GC.run            # force GC
```

**JFR (Java Flight Recorder)** — Low overhead profiling:
```bash
jcmd <pid> JFR.start duration=60s filename=recording.jfr
# Analyze với JDK Mission Control
```

---

## 2. Vấn đề thường gặp & Cách fix

### Issue 1: OutOfMemoryError: Java heap space

**Chẩn đoán:**
```bash
# 1. Enable auto heap dump on OOM
java -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heap.hprof MyApp

# 2. Sau khi OOM xảy ra, analyze heap.hprof với Eclipse MAT
# Mở: File → Open Heap Dump → Leak Suspects Report
# Hoặc Dominator Tree → xem objects chiếm nhiều memory nhất

# 3. jstat để theo dõi realtime
jstat -gc <pid> 2000
# Nếu OU (Old Used) tăng steadily mà không giảm → leak
```

**Fix:** Tìm root cause từ MAT → thường là static Map, cache không có eviction, ThreadLocal không clear.

### Issue 2: Full GC gây high latency

**Chẩn đoán:**
```bash
# Enable GC logging
java -Xlog:gc*:file=/tmp/gc.log:tags,time,uptime,level MyApp

# Tìm trong log: [Full GC ...] với pause time > 1s
```

**Fix checklist:**
- Tăng heap size (`-Xmx`) nếu OldGen luôn > 80%
- Switch sang ZGC: `-XX:+UseZGC` (Java 17+)
- Giảm object allocation rate (pool objects, reuse)
- Fix memory leak nếu Old Gen tăng liên tục

### Issue 3: Deadlock

**Chẩn đoán:**
```bash
jstack <pid> | grep -A 20 "deadlock"
# Output:
# Thread A: waiting for lock <0x00000000> (held by Thread B)
# Thread B: waiting for lock <0x00000001> (held by Thread A)
```

**Fix:** Luôn acquire locks theo cùng một thứ tự. Sử dụng `tryLock()` với timeout. Dùng higher-level abstractions như `java.util.concurrent`.

---

## 3. Code mẫu

```java
// MemoryLeakSimulator.java
import java.util.*;

public class MemoryLeakSimulator {

    // === Demo 1: Static cache leak ===
    // ❌ Leak: Map grow forever
    private static final Map<String, byte[]> leakingCache = new HashMap<>();

    public static void simulateLeak(int iterations) {
        System.out.println("Starting leak simulation...");
        for (int i = 0; i < iterations; i++) {
            // Mỗi entry: 1MB data, key không bao giờ remove
            leakingCache.put("key-" + i, new byte[1024 * 1024]);
            if (i % 100 == 0) {
                Runtime rt = Runtime.getRuntime();
                System.out.printf("Iteration %d: heap used %.0f MB%n",
                    i, (rt.totalMemory() - rt.freeMemory()) / 1e6);
            }
        }
    }

    // ✅ Fix: LRU cache với eviction
    private static final Map<String, byte[]> fixedCache = Collections.synchronizedMap(
        new LinkedHashMap<>(100, 0.75f, true) { // accessOrder=true
            @Override
            protected boolean removeEldestEntry(Map.Entry<String, byte[]> eldest) {
                return size() > 100; // max 100 entries
            }
        }
    );

    public static void simulateFixed(int iterations) {
        System.out.println("Starting fixed simulation (max 100 entries)...");
        for (int i = 0; i < iterations; i++) {
            fixedCache.put("key-" + i, new byte[1024 * 1024]);
            if (i % 100 == 0) {
                Runtime rt = Runtime.getRuntime();
                System.out.printf("Iteration %d: cache size=%d, heap used %.0f MB%n",
                    i, fixedCache.size(), (rt.totalMemory() - rt.freeMemory()) / 1e6);
            }
        }
    }
}
```

```java
// DeadlockDemo.java
public class DeadlockDemo {

    private static final Object LOCK_A = new Object();
    private static final Object LOCK_B = new Object();

    // ❌ Deadlock: Thread 1 acquires A then B; Thread 2 acquires B then A
    public static void demonstrateDeadlock() {
        Thread t1 = new Thread(() -> {
            synchronized (LOCK_A) {
                System.out.println("T1 acquired A, waiting for B...");
                try { Thread.sleep(100); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                synchronized (LOCK_B) { System.out.println("T1 acquired B"); }
            }
        }, "DeadlockThread-1");

        Thread t2 = new Thread(() -> {
            synchronized (LOCK_B) {  // ← different order!
                System.out.println("T2 acquired B, waiting for A...");
                try { Thread.sleep(100); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                synchronized (LOCK_A) { System.out.println("T2 acquired A"); }
            }
        }, "DeadlockThread-2");

        t1.start();
        t2.start();
        // Cả hai thread sẽ block mãi mãi
        // jstack sẽ report "Found 1 deadlock"
    }

    // ✅ Fix: consistent lock ordering
    public static void fixedTransfer() {
        // Always acquire lower-id lock first
        Thread t1 = new Thread(() -> {
            synchronized (LOCK_A) { // same order as t2
                synchronized (LOCK_B) { System.out.println("T1 done"); }
            }
        });

        Thread t2 = new Thread(() -> {
            synchronized (LOCK_A) { // same order
                synchronized (LOCK_B) { System.out.println("T2 done"); }
            }
        });

        t1.start();
        t2.start();
    }
}
```

```java
// StringConcatBenchmark.java — Why StringBuilder matters
public class StringConcatBenchmark {

    public static String concatWithPlus(int n) {
        String result = "";
        for (int i = 0; i < n; i++) {
            result += i; // Creates NEW String object each iteration → O(n²) total
        }
        return result;
    }

    // ✅ O(n) — single mutable buffer
    public static String concatWithBuilder(int n) {
        StringBuilder sb = new StringBuilder(n * 5); // pre-size hint
        for (int i = 0; i < n; i++) {
            sb.append(i);
        }
        return sb.toString();
    }

    public static void benchmark(int n) {
        System.out.println("Benchmarking with n=" + n);

        long start = System.currentTimeMillis();
        String r1 = concatWithPlus(n);
        System.out.println("String+: " + (System.currentTimeMillis() - start) + "ms, len=" + r1.length());

        start = System.currentTimeMillis();
        String r2 = concatWithBuilder(n);
        System.out.println("StringBuilder: " + (System.currentTimeMillis() - start) + "ms, len=" + r2.length());
    }

    public static void main(String[] args) {
        benchmark(10_000);
        benchmark(50_000);
    }
}
```

**JVM flags cheatsheet:**
```bash
# Heap sizing
-Xms512m           # initial heap (set equal to Xmx to avoid resize pauses)
-Xmx2g             # max heap
-XX:+UseContainerSupport   # auto-detect container memory limit

# GC selection
-XX:+UseG1GC               # default Java 9+
-XX:+UseZGC                # low-latency (Java 15+ prod-ready)
-XX:MaxGCPauseMillis=200   # G1 pause target

# Diagnostics
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/
-Xlog:gc*:file=/tmp/gc.log:tags,time
-XX:+PrintCompilation      # see JIT compilation

# Container-friendly
-XX:MaxRAMPercentage=75.0  # use 75% of container memory as max heap
```

---

## 4. Lab Steps

1. **Memory leak:** Chạy `simulateLeak(500)` → monitor với `jstat -gc <pid> 2000` → xem OU tăng
2. **Fix:** Switch sang `simulateFixed(500)` → cache size không vượt 100
3. **Deadlock:** Chạy `demonstrateDeadlock()` → để 3 giây → `jstack <pid>` → tìm "deadlock detected"
4. **String concat:** Chạy benchmark với n=50000 → String+ chậm hơn StringBuilder bao nhiêu lần?
5. **Thread dump reading:** Phân biệt `RUNNABLE` (đang chạy), `TIMED_WAITING` (sleep/wait timeout), `BLOCKED` (waiting for lock)
6. **GC tuning:** Add `-Xlog:gc*` flag → chạy app → đọc GC log, identify minor vs full GC

---

## 5. Checklist tự kiểm tra

- [ ] `jstat -gc` output: biết OU (Old Used) và FGCT (Full GC time) là gì
- [ ] Phân biệt Minor GC (Young Gen) và Full GC (toàn bộ heap)
- [ ] Memory leak: object held by GC root (static, ThreadLocal) không bao giờ collected
- [ ] Thread dump: `BLOCKED` = waiting for monitor/synchronized, `WAITING` = Object.wait/LockSupport.park
- [ ] `-XX:+UseContainerSupport` bắt buộc trong Docker — tránh JVM không thấy container memory limit
- [ ] String concatenation trong loop → `StringBuilder` — đặc biệt khi n > 1000
