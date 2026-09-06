# Phase 03 — Functional Java · Lý thuyết & Lab

---

## 1. Lý thuyết

### 1.0 Method References — 4 Loại

Method reference là shorthand cho lambda khi lambda chỉ delegate tới một method có sẵn. Cú pháp: `ClassName::methodName`.

**Loại 1 — Static method reference:**
```java
// Lambda:
Function<String, Integer> parser = s -> Integer.parseInt(s);
// Method reference:
Function<String, Integer> parser = Integer::parseInt;

// Cả hai tương đương. Method reference ngắn hơn, ý định rõ hơn.
List<String> nums = List.of("1", "2", "3");
List<Integer> ints = nums.stream().map(Integer::parseInt).toList();
```

**Loại 2 — Instance method reference của một object cụ thể:**
```java
String prefix = "Hello";
// Lambda:
Predicate<String> startsWith = s -> prefix.startsWith(s); // sử dụng captured variable
// Method reference:
Predicate<String> startsWith = prefix::startsWith;

// Thường gặp với System.out:
Consumer<String> printer = System.out::println; // bound to System.out instance
list.forEach(System.out::println);
```

**Loại 3 — Instance method reference của arbitrary object (của class):**
```java
// Lambda:
Function<String, String> toUpper = s -> s.toUpperCase();
// Method reference:
Function<String, String> toUpper = String::toUpperCase;
// Khác loại 2: không bound vào object cụ thể — đây là method trên bất kỳ String nào

// Với Comparator:
List<String> words = List.of("banana", "apple", "cherry");
words.sort(String::compareToIgnoreCase); // s1.compareToIgnoreCase(s2)
```

**Loại 4 — Constructor reference:**
```java
// Lambda:
Supplier<ArrayList<String>> listFactory = () -> new ArrayList<>();
// Constructor reference:
Supplier<ArrayList<String>> listFactory = ArrayList::new;

// Với BiFunction:
BiFunction<Integer, Integer, int[]> arrayCreator = int[]::new; // creates int[n][m]? No, int array
// Thực tế hơn:
Function<Integer, int[]> intArrayMaker = int[]::new; // n -> new int[n]
```

**Chọn dạng nào:**
- Lambda khi logic cần transform/compose
- Method reference khi "đây chỉ là method A trên type B" — ý định rõ hơn

---

### 1.00 BiFunction, UnaryOperator, BinaryOperator

**Functional interface mở rộng từ Function:**

```java
// BiFunction<T, U, R> — nhận 2 args, trả 1 result
BiFunction<String, Integer, String> repeat = (s, n) -> s.repeat(n);
String result = repeat.apply("ha", 3); // "hahaha"

// Combine với andThen:
BiFunction<String, Integer, String> repeatAndUpper =
    repeat.andThen(String::toUpperCase); // "HAHAHA"
```

```java
// UnaryOperator<T> — Function<T,T> — input và output cùng type
UnaryOperator<String> trim = String::trim;
UnaryOperator<String> upper = String::toUpperCase;

// Chain với andThen / compose:
UnaryOperator<String> normalize = trim.andThen(upper)::apply; // trim THEN upper
// Chú ý: UnaryOperator không có andThen trả UnaryOperator — phải cast:
Function<String, String> normalize2 = trim.andThen(upper);

// iterate() với UnaryOperator:
Stream<Integer> powers = Stream.iterate(1, n -> n * 2); // 1, 2, 4, 8, ...
```

```java
// BinaryOperator<T> — BiFunction<T,T,T> — 2 args cùng type, trả cùng type
BinaryOperator<Integer> sum = Integer::sum;
BinaryOperator<String> concat = String::concat;

// Dùng với reduce:
Optional<Integer> total = Stream.of(1, 2, 3, 4).reduce(Integer::sum);
int totalWithIdentity = Stream.of(1, 2, 3, 4).reduce(0, Integer::sum);

// BinaryOperator.maxBy/minBy:
BinaryOperator<String> longerStr = BinaryOperator.maxBy(Comparator.comparingInt(String::length));
```

**Tóm tắt:**
| Interface | Signature | Dùng khi |
|---|---|---|
| `Function<T,R>` | `T → R` | transform một value |
| `BiFunction<T,U,R>` | `T,U → R` | combine hai values |
| `UnaryOperator<T>` | `T → T` | transform giữ nguyên type |
| `BinaryOperator<T>` | `T,T → T` | combine hai values cùng type |
| `Predicate<T>` | `T → boolean` | filter/test |
| `Consumer<T>` | `T → void` | side effects |
| `Supplier<T>` | `() → T` | factory/lazy |

---

### 1.000 Pure Function & Side Effects

**Pure function — không có side effects:**
1. Với cùng input, luôn trả cùng output (deterministic)
2. Không modify state bên ngoài (no side effects)

```java
// ✅ Pure function
int add(int a, int b) { return a + b; } // cùng args → luôn cùng kết quả

// ❌ Impure — đọc external state
int multiplyByFactor(int n) { return n * this.factor; } // kết quả phụ thuộc this.factor

// ❌ Impure — side effect: modify external state
void addToList(int n) { sharedList.add(n); } // thay đổi shared state

// ❌ Impure — I/O là side effect
String readFile(String path) throws IOException { return Files.readString(Path.of(path)); }
```

**Tại sao pure functions quan trọng trong Streams:**
- **Thread-safe:** Parallel streams có thể chạy nhiều threads cùng lúc. Pure functions không share mutable state → safe.
- **Testable:** Input/output rõ ràng, không cần mock.
- **Composable:** `f.andThen(g)` chỉ safe nếu cả `f` và `g` là pure.

**Danger: stateful lambdas trong stream:**
```java
// ❌ DANGEROUS — stateful lambda: violates stream contract
List<Integer> seen = new ArrayList<>(); // mutable external state
Stream.of(1, 2, 3, 1, 2).filter(n -> !seen.contains(n) && seen.add(n)).toList();
// sequential: works accidentally
// parallel: broken — concurrent access to `seen`

// ✅ Correct: use distinct() for deduplication
Stream.of(1, 2, 3, 1, 2).distinct().toList();

// ✅ Correct: if you need custom logic, collect first then process
```

**Referential transparency:** Pure function có thể được replaced bởi return value của nó mà không thay đổi behavior của program. Điều này enable memoization, lazy evaluation, và compiler optimization.

---

### 1.1 Lambda & Functional Interface

**Lambda là gì:**
Lambda là một **anonymous function** — có thể được truyền như giá trị, lưu trong biến, trả về từ method. Java implement bằng cách tự động implement single-abstract-method (SAM) interface.

```java
// Trước Java 8: anonymous class — verbose
Comparator<String> comp = new Comparator<String>() {
    @Override
    public int compare(String a, String b) { return a.length() - b.length(); }
};

// Sau: lambda — gọn, không cần boilerplate
Comparator<String> comp = (a, b) -> a.length() - b.length();

// Method reference — khi lambda chỉ gọi một method có sẵn
Comparator<String> comp = Comparator.comparingInt(String::length);
```

**Bốn functional interface cốt lõi:**
| Interface | Method | Mô tả |
|---|---|---|
| `Function<T,R>` | `R apply(T t)` | Nhận T, trả R |
| `Predicate<T>` | `boolean test(T t)` | Nhận T, trả boolean |
| `Consumer<T>` | `void accept(T t)` | Nhận T, không trả |
| `Supplier<T>` | `T get()` | Không nhận, trả T |

**Composing predicates:**
```java
Predicate<Integer> positive = n -> n > 0;
Predicate<Integer> even = n -> n % 2 == 0;

// AND: cả hai điều kiện
Predicate<Integer> positiveEven = positive.and(even);

// OR: một trong hai
Predicate<Integer> positiveOrEven = positive.or(even);

// NOT
Predicate<Integer> notPositive = positive.negate();
```

---

### 1.2 Stream API — Lazy Pipeline

**Stream không phải Collection:**
- Collection: stores data, eager
- Stream: pipeline of operations, **lazy** (không compute gì cho đến khi có terminal operation)

**Lazy evaluation — tại sao quan trọng:**
```java
// Không có gì chạy ở đây — chỉ build pipeline
Stream<String> stream = names.stream()
    .filter(s -> { System.out.println("filter: " + s); return s.startsWith("A"); })
    .map(s -> { System.out.println("map: " + s); return s.toUpperCase(); });

// Terminal operation kích hoạt pipeline
List<String> result = stream.collect(Collectors.toList());
// Filter và map chạy xen kẽ: filter "Alice" → map "ALICE" → filter "Bob" → skip
```

**Short-circuit operations:** `findFirst()`, `anyMatch()`, `limit()` có thể dừng pipeline sớm khi đã có kết quả. Với `filter + findFirst`, không cần duyệt hết list.

**Intermediate vs Terminal:**
- Intermediate (lazy): `filter`, `map`, `flatMap`, `sorted`, `distinct`, `limit`, `skip`
- Terminal (triggers execution): `collect`, `forEach`, `count`, `findFirst`, `anyMatch`, `reduce`, `toList()`

**Stateful vs Stateless operations:**
- Stateless: `filter`, `map` — xử lý từng element độc lập, tốt cho parallel
- Stateful: `sorted`, `distinct` — cần thấy tất cả elements trước khi output bất kỳ, khó parallelize

---

### 1.3 Collectors — Tổng hợp kết quả

**`groupingBy`:**
```java
// Group transactions by category → Map<Category, List<Transaction>>
Map<Category, List<Transaction>> byCategory = transactions.stream()
    .collect(Collectors.groupingBy(Transaction::category));

// Group with downstream collector: count per category
Map<Category, Long> countByCategory = transactions.stream()
    .collect(Collectors.groupingBy(Transaction::category, Collectors.counting()));

// Group with sum
Map<Category, Double> sumByCategory = transactions.stream()
    .collect(Collectors.groupingBy(
        Transaction::category,
        Collectors.summingDouble(Transaction::amount)
    ));
```

**Custom Collector với `Collector.of()`:**
Custom collector cần 4 phần:
- `supplier`: tạo container mới (ví dụ `HashMap::new`)
- `accumulator`: thêm một element vào container
- `combiner`: gộp 2 containers (cho parallel stream)
- `finisher`: transform container thành kết quả cuối

```java
// Custom collector: group by YearMonth → sum amounts
Collector<Transaction, ?, Map<YearMonth, Double>> monthlyTotals = Collector.of(
    HashMap::new,                                          // supplier
    (map, t) -> map.merge(                                 // accumulator
        YearMonth.from(t.date()), t.amount(), Double::sum
    ),
    (map1, map2) -> { map2.forEach((k,v) -> map1.merge(k, v, Double::sum)); return map1; }, // combiner
    map -> Collections.unmodifiableMap(new TreeMap<>(map)) // finisher: sort by date
);
```

---

### 1.4 Optional — Null Safety

**Tại sao Optional:**
`null` gây ra NPE ở runtime — không có compile-time warning. Optional làm cho "có thể absent" trở thành **explicit type-level contract**. Caller biết phải xử lý empty case.

**Optional là monad:** Chain operations mà không phải kiểm tra null ở mỗi bước:
```java
// Cascade của null checks — verbose và error-prone
User user = getUser(id);
if (user != null) {
    Address address = user.getAddress();
    if (address != null) {
        String city = address.getCity();
        if (city != null) return city.toUpperCase();
    }
}
return "UNKNOWN";

// Optional — chain các transformations
return getUser(id)
    .map(User::getAddress)
    .map(Address::getCity)
    .map(String::toUpperCase)
    .orElse("UNKNOWN");
```

**Anti-patterns:**
```java
// ❌ isPresent() + get() = verbose null check (worse than original null)
if (opt.isPresent()) return opt.get().process();

// ❌ Optional field trong class (Optional không Serializable, verbose)
class User { Optional<String> phone; } // BAD

// ❌ Optional trong Collection (dư thừa — dùng filter thay)
List<Optional<String>> list; // BAD — dùng List<String> và filter

// ✅ Correct uses
return opt.map(User::process).orElseThrow(() -> new NotFoundException("..."));
return opt.ifPresent(u -> log.info("Found: {}", u));
```

---

### 1.5 Parallel Streams — Khi nào hiệu quả

**Parallel stream hoạt động bằng ForkJoinPool:**
Chia stream thành subtasks, execute song song trên CPU cores, merge kết quả. `Runtime.getRuntime().availableProcessors()` cores được dùng.

**Khi parallel stream CHẬM hơn sequential:**
1. **Overhead quá lớn so với work:** Với 100 elements đơn giản, overhead fork/join > time saved
2. **Stateful operations:** `sorted`, `distinct` cần synchronize
3. **Non-splittable sources:** LinkedList không splittable hiệu quả (ArrayList, array tốt hơn)
4. **IO-bound:** Parallel chỉ giúp CPU-bound work

**Rule of thumb:** Dataset lớn (> 10K elements) + CPU-intensive operations + splittable source (ArrayList, array, range) → dùng parallel.

---

## 2. Vấn đề thường gặp & Cách fix

### Issue 1: Side effects trong Stream

```java
// ❌ Mutating external state trong lambda — không safe với parallel
List<String> results = new ArrayList<>();
stream.filter(pred).forEach(s -> results.add(s)); // race condition trong parallel!

// ✅ Collect — stream manages accumulation
List<String> results = stream.filter(pred).collect(Collectors.toList());
```

### Issue 2: Stream đã consumed

```java
// ❌ Stream chỉ dùng được 1 lần
Stream<String> stream = list.stream().filter(s -> s.startsWith("A"));
long count = stream.count();     // terminal — stream consumed
List<String> list = stream.toList(); // IllegalStateException: stream already operated upon or closed!

// ✅ Tạo stream mới mỗi lần, hoặc collect trước
List<String> filtered = list.stream().filter(...).collect(Collectors.toList());
long count = filtered.size();
```

### Issue 3: `flatMap` vs `map`

```java
// map trả Stream<Stream<String>> — không phải điều bạn muốn
Stream<Stream<String>> nested = sentences.stream()
    .map(s -> Arrays.stream(s.split(" ")));

// flatMap "flatten" inner stream
Stream<String> words = sentences.stream()
    .flatMap(s -> Arrays.stream(s.split(" ")));
```

---

## 3. Code mẫu — Data Processing Engine

```java
// domain/Transaction.java
public record Transaction(
    String id,
    String userId,
    double amount,
    Category category,
    LocalDate date,
    String description
) {
    public enum Category { FOOD, TRANSPORT, ENTERTAINMENT, HEALTH, UTILITIES, OTHER }

    public static Transaction random() {
        var categories = Category.values();
        return new Transaction(
            UUID.randomUUID().toString().substring(0, 8),
            "user-" + (int)(Math.random() * 100),
            Math.round(Math.random() * 500_000) / 100.0,
            categories[(int)(Math.random() * categories.length)],
            LocalDate.now().minusDays((int)(Math.random() * 90)),
            "Sample transaction"
        );
    }
}
```

```java
// processing/TransactionProcessor.java
public class TransactionProcessor {

    private final List<Transaction> transactions;

    public TransactionProcessor(List<Transaction> transactions) {
        this.transactions = transactions;
    }

    // === Predicate composition ===
    public List<Transaction> filter(Predicate<Transaction> predicate) {
        return transactions.stream()
            .filter(predicate)
            .collect(Collectors.toList());
    }

    public static Predicate<Transaction> amountBetween(double min, double max) {
        return t -> t.amount() >= min && t.amount() <= max;
    }

    public static Predicate<Transaction> inCategory(Transaction.Category... categories) {
        Set<Transaction.Category> set = Set.of(categories);
        return t -> set.contains(t.category());
    }

    public static Predicate<Transaction> inLastDays(int days) {
        LocalDate cutoff = LocalDate.now().minusDays(days);
        return t -> t.date().isAfter(cutoff);
    }

    // === Statistics per category ===
    public Map<Transaction.Category, DoubleSummaryStatistics> statsByCategory() {
        return transactions.stream()
            .collect(Collectors.groupingBy(
                Transaction::category,
                Collectors.summarizingDouble(Transaction::amount)
            ));
    }

    // === Top N spenders ===
    public List<Map.Entry<String, Double>> topSpenders(int n) {
        return transactions.stream()
            .collect(Collectors.groupingBy(
                Transaction::userId,
                Collectors.summingDouble(Transaction::amount)
            ))
            .entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .limit(n)
            .collect(Collectors.toList());
    }

    // === Partition by threshold ===
    public Map<Boolean, List<Transaction>> partitionByAmount(double threshold) {
        return transactions.stream()
            .collect(Collectors.partitioningBy(t -> t.amount() >= threshold));
    }

    // === Custom Collector: monthly totals ===
    public Map<YearMonth, Double> monthlyTotals() {
        return transactions.stream().collect(
            Collector.of(
                TreeMap::new,   // sorted by key
                (map, t) -> map.merge(YearMonth.from(t.date()), t.amount(), Double::sum),
                (m1, m2) -> { m2.forEach((k, v) -> m1.merge(k, v, Double::sum)); return m1; }
            )
        );
    }

    // === Parallel benchmark ===
    public static void benchmarkSequentialVsParallel(List<Transaction> data) {
        // Warmup
        for (int i = 0; i < 3; i++) {
            data.stream().mapToDouble(Transaction::amount).sum();
            data.parallelStream().mapToDouble(Transaction::amount).sum();
        }

        long start = System.nanoTime();
        double seqSum = data.stream()
            .filter(t -> t.amount() > 100)
            .mapToDouble(Transaction::amount)
            .sum();
        long seqTime = System.nanoTime() - start;

        start = System.nanoTime();
        double parSum = data.parallelStream()
            .filter(t -> t.amount() > 100)
            .mapToDouble(Transaction::amount)
            .sum();
        long parTime = System.nanoTime() - start;

        System.out.printf("Sequential: %.2f ms | Parallel: %.2f ms | Ratio: %.2fx%n",
            seqTime / 1e6, parTime / 1e6, (double) seqTime / parTime);
    }
}
```

---

## 4. Lab Steps

1. **Lazy evaluation:** Thêm `System.out.println` vào filter lambda, xem output thứ tự khi dùng `findFirst` vs `collect`
2. **Predicate composition:** Tạo filter: giao dịch > 100K VND, trong 30 ngày qua, category FOOD hoặc TRANSPORT
3. **GroupingBy downstream:** So sánh output của `groupingBy(cat)` vs `groupingBy(cat, counting())`
4. **Custom Collector:** Implement `monthlyTotals()`, verify tổng bằng sum tất cả transactions
5. **Parallel benchmark:** Chạy với 100K transactions — xem parallel có nhanh hơn không
6. **Optional chain:** Viết method tìm giao dịch lớn nhất của một user, return `Optional<Double>`

---

## 5. Checklist tự kiểm tra

- [ ] `filter + findFirst` chỉ duyệt đến phần tử đầu tiên thỏa điều kiện (verify bằng println)
- [ ] `flatMap` vs `map`: biết khi nào mỗi cái được dùng
- [ ] `Collectors.groupingBy` với downstream collector (counting, summingDouble, toList)
- [ ] Custom Collector: combiner được gọi trong parallel stream, không phải sequential
- [ ] Optional không dùng `isPresent() + get()` — dùng `map`, `orElse`, `ifPresent`
- [ ] Parallel stream không safe với mutable shared state — luôn collect thay vì forEach+add
