# Phase 06 — JVM, Memory & Performance

> **Mục tiêu:** Hiểu JVM đủ sâu để debug memory leak, GC pause, CPU spike — không phải để build JVM.

---

## Part 1 — JVM Architecture

```
┌─────────────────────────────────────────────────────┐
│                     JVM                             │
│                                                     │
│  ┌────────────┐  ┌──────────┐  ┌─────────────────┐ │
│  │ Class      │  │ Runtime  │  │ Execution       │ │
│  │ Loader     │  │ Data     │  │ Engine          │ │
│  │ Subsystem  │  │ Area     │  │                 │ │
│  └────────────┘  └──────────┘  │  ┌───────────┐  │ │
│                                │  │Interpreter│  │ │
│                                │  ├───────────┤  │ │
│                                │  │JIT        │  │ │
│                                │  │Compiler   │  │ │
│                                │  └───────────┘  │ │
│                                └─────────────────┘ │
└─────────────────────────────────────────────────────┘
```

### Class Loader
```
Bootstrap ClassLoader    ← load java.lang.*, java.util.*
        ↓
Extension ClassLoader    ← load $JAVA_HOME/lib/ext
        ↓
Application ClassLoader  ← load classpath của bạn
```
**Parent delegation model:** Trước khi load class, delegate cho parent trước. Tại sao: bảo vệ core classes, tránh duplicate loading.

---

## Part 2 — Memory Model

```
┌────────────────────────────────────────────────────┐
│                      Heap                          │
│  ┌─────────────────────┐  ┌──────────────────────┐ │
│  │   Young Generation  │  │    Old Generation    │ │
│  │  ┌────┐ ┌──┐ ┌──┐  │  │                      │ │
│  │  │Eden│ │S0│ │S1│  │  │  Long-lived objects  │ │
│  │  └────┘ └──┘ └──┘  │  │                      │ │
│  └─────────────────────┘  └──────────────────────┘ │
└────────────────────────────────────────────────────┘
┌───────────┐  ┌────────────┐  ┌────────────────────┐
│  Stack    │  │  Metaspace │  │    Code Cache      │
│ (per      │  │ (class     │  │  (JIT compiled     │
│  thread)  │  │  metadata) │  │   code)            │
└───────────┘  └────────────┘  └────────────────────┘
```

| Area | Stores | Error khi đầy |
|------|--------|---------------|
| Heap | Objects, arrays | `OutOfMemoryError: Java heap space` |
| Stack | Local variables, frames | `StackOverflowError` |
| Metaspace | Class metadata | `OutOfMemoryError: Metaspace` |
| Code Cache | JIT compiled code | Không JIT, performance drop |

---

## Part 3 — Garbage Collection

### Object Lifecycle
```
new Object()
    ↓
Eden space → Minor GC → S0/S1 (Survivor)
    ↓ (survive N GCs)
Old Generation → Major GC / Full GC
```

### GC Algorithms
| GC | Đặc điểm | JVM flag | Dùng khi |
|----|---------|---------|---------|
| Serial | Single thread | `-XX:+UseSerialGC` | Single-core, small heap |
| Parallel | Multi-thread, STW | `-XX:+UseParallelGC` | Throughput priority |
| G1 | Region-based, concurrent | `-XX:+UseG1GC` | Default Java 9+, balanced |
| ZGC | Concurrent, sub-ms pause | `-XX:+UseZGC` | Low-latency, large heap (Java 15+) |
| Shenandoah | Concurrent, low pause | `-XX:+UseShenandoahGC` | OpenJDK, similar to ZGC |

**STW = Stop The World** — tất cả application threads dừng lại.

### GC Tuning cơ bản
```bash
-Xms2g -Xmx2g          # Set heap size (min = max để tránh resize)
-XX:NewRatio=3          # Old:Young = 3:1
-XX:MaxGCPauseMillis=200 # G1 target pause time
-XX:+PrintGCDetails     # Log GC events
-Xlog:gc*:file=gc.log   # Java 11+ GC log
```

---

## Part 4 — JIT Compilation

```
Bytecode → Interpreter (chậm)
              ↓ (hot code detected)
         JIT Compiler → Native code (nhanh)
```

**Tiered Compilation:**
- Level 0: Interpreter
- Level 1-3: Client compiler (C1) — nhanh compile, ít optimize
- Level 4: Server compiler (C2) — chậm compile, optimize mạnh

**-XX:CompileThreshold=10000** — method được JIT sau 10k lần gọi.

---

## Part 5 — Diagnostic Tools

### Command-line tools
```bash
jps                          # list Java processes
jstack <pid>                 # thread dump
jmap -heap <pid>             # heap summary
jmap -dump:format=b,file=heap.hprof <pid>  # heap dump
jstat -gc <pid> 1000         # GC stats mỗi 1 giây
jcmd <pid> VM.flags          # JVM flags đang dùng
jcmd <pid> Thread.print      # thread dump (thay jstack)
jcmd <pid> GC.heap_info      # heap info
jcmd <pid> JFR.start         # bắt đầu Java Flight Recorder
```

### Visual Tools
| Tool | Dùng để | Cách mở |
|------|---------|---------|
| VisualVM | Monitor heap, CPU, threads real-time | `visualvm` hoặc download |
| JFR + JMC | Low-overhead profiling, production-safe | `-XX:+FlightRecorder` |
| Eclipse MAT | Phân tích heap dump, tìm leak | Import `.hprof` |

---

## Project — JVM Performance Lab

**Mục tiêu: Cố tình tạo vấn đề, sau đó diagnose và fix.**

```
src/
├── scenarios/
│   ├── MemoryLeakSimulator.java      ← static collection grow forever
│   ├── HighAllocationRate.java       ← tạo nhiều short-lived objects
│   ├── GcPressureDemo.java           ← trigger frequent GC
│   ├── ThreadExplosionDemo.java      ← tạo quá nhiều threads
│   ├── StackOverflowDemo.java        ← recursion không có base case
│   └── ClassLoaderLeakDemo.java      ← dynamic class loading leak
├── diagnosis/
│   ├── HeapAnalysis.md               ← note cách đọc heap dump
│   ├── ThreadDumpAnalysis.md         ← note cách đọc thread dump
│   └── GcLogAnalysis.md              ← note cách đọc GC log
└── benchmark/
    ├── StringConcatBenchmark.java    ← String vs StringBuilder performance
    └── CollectionBenchmark.java      ← ArrayList vs LinkedList vs ArrayDeque
```

**Workflow cho mỗi scenario:**
1. Chạy scenario
2. Observe triệu chứng (high memory, high CPU, freeze)
3. Capture thread dump / heap dump
4. Analyze
5. Identify root cause
6. Fix
7. Verify fix

---

## Checklist — 6 câu hỏi

Ví dụ với `Memory Leak`:

| # | Câu hỏi | Trả lời |
|---|---------|---------|
| 1 | Là gì? | Object không còn được dùng nhưng vẫn có reference → GC không collect được |
| 2 | Giải quyết gì? | N/A — đây là vấn đề cần tránh |
| 3 | Hoạt động thế nào? | Static collection giữ reference → object không bao giờ unreachable |
| 4 | Khi nào xảy ra? | Static cache không có eviction, listener không unregister, ThreadLocal không clear |
| 5 | Cách tránh | WeakReference cho cache; unregister listeners; clear ThreadLocal trong finally |
| 6 | Debug thế nào? | jstat theo dõi heap growth; heap dump + MAT → Dominator Tree → tìm root cause |

---

## Run

```bash
mvn compile -pl phase-06-jvm-memory-performance

# Chạy với GC logging
java -Xmx256m -Xlog:gc*:file=gc.log \
     -cp target/classes com.nguyenngoc.phase06.scenarios.HighAllocationRate

# Chạy với JFR
java -XX:+FlightRecorder \
     -XX:StartFlightRecording=duration=60s,filename=recording.jfr \
     -cp target/classes com.nguyenngoc.phase06.scenarios.MemoryLeakSimulator
```
