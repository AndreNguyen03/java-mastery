# Phase 03 — Functional Java

> **Mục tiêu:** Không chỉ biết `.stream()`. Hiểu khi nào Stream tốt, khi nào xấu, và tại sao — từ đó viết code functional đúng chỗ, không phải mọi chỗ.

---

## Part 1 — Lambda & Functional Interface

### Lambda
```java
// Anonymous class (verbose)
Comparator<String> c1 = new Comparator<String>() {
    public int compare(String a, String b) { return a.compareTo(b); }
};

// Lambda
Comparator<String> c2 = (a, b) -> a.compareTo(b);

// Method reference
Comparator<String> c3 = String::compareTo;
```

Lambda **không phải** anonymous class — JVM xử lý khác nhau (invokedynamic vs new class file).

### Functional Interfaces — Bộ tứ cơ bản
| Interface | Signature | Dùng khi |
|-----------|-----------|---------|
| `Function<T, R>` | `T → R` | Transform một giá trị |
| `Predicate<T>` | `T → boolean` | Filter, test condition |
| `Consumer<T>` | `T → void` | Consume, side effect |
| `Supplier<T>` | `() → T` | Provide, lazy evaluation |
| `BiFunction<T,U,R>` | `(T,U) → R` | 2 inputs → 1 output |
| `UnaryOperator<T>` | `T → T` | Transform same type |
| `BinaryOperator<T>` | `(T,T) → T` | Combine same type |

### Method Reference — 4 loại
```java
String::toUpperCase          // instance method (unbound)
"hello"::toUpperCase         // instance method (bound)
String::new                  // constructor reference
Integer::parseInt            // static method
```

---

## Part 2 — Stream API

### Pipeline structure
```
Source → [Intermediate ops]* → Terminal op
                                     ↑
                          Trigger lazy evaluation
```

Stream là **lazy**: intermediate operations không chạy cho đến khi có terminal.

### Intermediate Operations
| Op | Mô tả | Notes |
|----|-------|-------|
| `filter(Predicate)` | Lọc phần tử | Stateless |
| `map(Function)` | Transform từng phần tử | Stateless |
| `flatMap(Function)` | Flatten nested collections | Stateless |
| `distinct()` | Loại trùng | Stateful — cần buffer |
| `sorted()` | Sắp xếp | Stateful — cần thấy toàn bộ |
| `limit(n)` | Lấy n phần tử đầu | Short-circuit |
| `skip(n)` | Bỏ n phần tử đầu | Stateful |
| `peek(Consumer)` | Debug side effect | Không dùng trong production |

### Terminal Operations
| Op | Mô tả | Trả về |
|----|-------|-------|
| `collect(Collector)` | Thu thập vào collection | `R` |
| `forEach(Consumer)` | Lặp với side effect | `void` |
| `reduce(identity, BinaryOperator)` | Fold thành một giá trị | `T` |
| `count()` | Đếm | `long` |
| `findFirst()` / `findAny()` | Tìm phần tử | `Optional<T>` |
| `anyMatch` / `allMatch` / `noneMatch` | Test | `boolean` |
| `min()` / `max()` | Cực trị | `Optional<T>` |
| `toList()` (Java 16+) | Shortcut | `List<T>` (unmodifiable) |

### Collectors quan trọng
```java
// Group by
Map<Category, List<Expense>> byCategory =
    expenses.stream()
        .collect(Collectors.groupingBy(Expense::category));

// Partition
Map<Boolean, List<Expense>> partition =
    expenses.stream()
        .collect(Collectors.partitioningBy(e -> e.amount() > 1000));

// Statistics
IntSummaryStatistics stats =
    expenses.stream()
        .mapToInt(Expense::amount)
        .summaryStatistics();

// Joining
String csv = names.stream().collect(Collectors.joining(", "));

// Downstream collector
Map<Category, Long> countByCategory =
    expenses.stream()
        .collect(Collectors.groupingBy(Expense::category, Collectors.counting()));
```

---

## Part 3 — Optional

### Mục đích
Không phải "tránh null pointer". Mà là **model hóa sự vắng mặt** — buộc caller phải xử lý trường hợp không có giá trị.

```java
// Sai: dùng Optional như null check thông thường
if (optional.isPresent()) { optional.get(); }

// Đúng: chain
Optional.ofNullable(findUser(id))
    .map(User::getProfile)
    .map(Profile::getAvatar)
    .orElse(DEFAULT_AVATAR);
```

**Khi nào KHÔNG dùng Optional:**
- Field của class (serialization vấn đề)
- Method parameter
- Collection element (dùng empty collection thay)

---

## Part 4 — Parallel Stream

```java
list.parallelStream().filter(...).map(...).collect(...)
```

**Parallel Stream không phải lúc nào cũng nhanh hơn.**

Overhead bao gồm: fork/join, merge, thread coordination.

| Dùng Parallel Stream khi | Không dùng khi |
|--------------------------|----------------|
| Dataset lớn (> 10k elements) | Dataset nhỏ |
| CPU-intensive (không I/O) | I/O bound operations |
| Stateless, no shared state | Stateful, ordered |
| Splittable source (ArrayList) | Non-splittable (LinkedList) |

**ForkJoinPool.commonPool()** — đây là pool mà parallel stream dùng, shared với toàn JVM.

---

## Part 5 — Functional Concepts

### Immutability
- Object không thay đổi sau khi tạo
- Thread-safe by design
- Dễ cache, dễ test
- Java: `final` fields + no setters, hoặc dùng `record`

### Pure Function
```
f(x) = y
- Cùng input → cùng output (deterministic)
- Không side effect (không thay đổi state ngoài)
```

**Tại sao quan trọng:** Dễ test, dễ optimize, dễ parallelize.

### Side Effect
Bất cứ thứ gì ngoài return value: modify field, I/O, print, DB call.

Stream intermediate ops nên **không có side effect** — `peek()` là exception cho debug, không phải production.

---

## Project — Data Processing Engine

**1,000,000 transaction records → filter → group → aggregate → statistics**

```
src/
├── model/
│   ├── Transaction.java         (Record)
│   ├── Category.java            (Enum)
│   └── Report.java              (Record)
├── generator/
│   └── TransactionGenerator.java   ← tạo 1M random transactions
├── processor/
│   ├── TransactionFilter.java      ← Predicate composition
│   ├── TransactionGrouper.java     ← groupingBy, partitioningBy
│   ├── TransactionAggregator.java  ← reduce, statistics
│   └── ReportBuilder.java          ← compose pipeline
├── benchmark/
│   ├── SequentialVsParallel.java   ← so sánh sequential vs parallel
│   └── StreamVsLoop.java           ← stream vs for loop benchmark
└── Main.java
```

**Yêu cầu:**
1. Sequential pipeline xử lý đúng kết quả
2. Parallel pipeline so sánh performance
3. Benchmark: stream vs traditional for-loop cho dataset nhỏ/lớn
4. Custom Collector: `Collector<Transaction, ?, MonthlyReport>`

---

## Checklist — 6 câu hỏi

Ví dụ với `Stream`:

| # | Câu hỏi | Trả lời |
|---|---------|---------|
| 1 | Là gì? | Sequence of elements supporting sequential and parallel aggregate operations |
| 2 | Giải quyết gì? | Xử lý collection theo kiểu declarative, không cần vòng lặp thủ công |
| 3 | Hoạt động thế nào? | Lazy pipeline: intermediate ops không chạy đến khi có terminal op |
| 4 | Khi nào dùng? | Transform/filter/aggregate collections, đặc biệt khi cần compose nhiều bước |
| 5 | Khi nào không? | Cần index, cần break sớm phức tạp, dataset nhỏ với loop đơn giản hơn |
| 6 | Debug thế nào? | `peek()` giữa các bước; nếu parallel → kiểm tra shared state, ordered |

---

## Run

```bash
mvn test -pl phase-03-functional-java
```
