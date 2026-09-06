# Phase 01 — Java Language · Lý thuyết & Lab

---

## 1. Lý thuyết nền tảng

### 1.0 Class, Interface & OOP Fundamentals

**Access Modifiers — tại sao quan trọng:**
- `private`: chỉ trong class → encapsulation, thay đổi nội bộ không ảnh hưởng callers
- `package-private` (default): trong cùng package → internal implementation detail
- `protected`: subclass access → template method pattern
- `public`: API contract → thay đổi là breaking change

**`static` vs instance:**
```java
class Counter {
    static int total = 0;   // shared by ALL instances — class-level state
    int count = 0;           // per-instance state

    void increment() {
        count++;    // this instance's count
        total++;    // ALL counters' total — shared!
    }
}
```
`static` method không có `this` reference — không thể access instance fields. Dùng cho utility methods (không cần state) hoặc factory methods.

**`final` keyword:**
- `final` class: không thể subclass (`String`, `Integer` là final — safe to cache/share)
- `final` method: không thể override — prevents unintended behavior change
- `final` field: phải được gán một lần (constructor hoặc initializer) — immutability building block

**Interface vs Abstract Class:**
| | Interface | Abstract Class |
|---|---|---|
| Constructor | Không | Có |
| Fields | Constants only (`public static final`) | Any fields |
| Methods | default, static, private (Java 9+) | Any |
| Extends | Nhiều interfaces | 1 class |
| Use when | Capability/behavior contract | Shared implementation + state |

**Interface default methods (Java 8+) — backward compatibility:**
```java
interface Drawable {
    void draw();
    
    default void drawWithBorder() {  // default: implementing classes don't need to override
        System.out.println("Border");
        draw();
    }
    
    static Drawable noop() { return () -> {}; } // factory
}
```
`default` methods giúp thêm methods vào interface mà không break existing implementations — key cho Java 8 evolution.

---

### 1.00 Generics — Type Safety Without Casting

**Tại sao Generics:**
Trước Java 5: `List` chứa `Object` → phải cast khi lấy ra → `ClassCastException` ở runtime.
```java
List list = new ArrayList();
list.add("hello");
list.add(42);
String s = (String) list.get(1); // ClassCastException! 42 is not String
```

Sau Java 5: Generic type check tại compile time:
```java
List<String> list = new ArrayList<>();
list.add("hello");
// list.add(42); // Compile error — caught at compile time, not runtime!
String s = list.get(0); // no cast needed
```

**Type Erasure — điều quan trọng nhất về Generics:**
Generic type information chỉ tồn tại ở **compile time**. JVM không biết về generics — bytecode chỉ có raw types. Compiler xóa type params và thêm casts.

```java
// Source code:
List<String> names = new ArrayList<>();
names.add("Alice");
String first = names.get(0);

// After erasure (bytecode equivalent):
List names = new ArrayList();
names.add("Alice");
String first = (String) names.get(0); // compiler adds cast
```

**Hệ quả của Type Erasure:**
```java
// Không thể tạo generic array
T[] arr = new T[10]; // Compile error

// Không thể dùng instanceof với generic
if (list instanceof List<String>) {} // Compile error

// Không thể overload chỉ khác type param (same after erasure)
void process(List<String> list) {}  // Both become: void process(List list)
void process(List<Integer> list) {} // Compile error: same erasure!
```

**Wildcards — Khi nào dùng gì:**
```java
// ? extends T (covariant — upper bounded) — "Producer Extends"
// Đọc được, không ghi được
List<? extends Number> numbers; // Can hold List<Integer>, List<Double>
Number n = numbers.get(0);      // ✅ Safe read
numbers.add(42);                // ❌ Compile error — unknown exact type

// ? super T (contravariant — lower bounded) — "Consumer Super"
// Ghi được, đọc ra Object
List<? super Integer> ints;  // Can hold List<Integer>, List<Number>, List<Object>
ints.add(42);                // ✅ Safe write
Object o = ints.get(0);     // Only Object guaranteed

// PECS Rule (Producer Extends, Consumer Super):
<T> void copy(List<? extends T> source,  // produces T: extends
              List<? super T> dest) {    // consumes T: super
    for (T t : source) dest.add(t);
}
```

**Bounded type parameters:**
```java
// T phải implement Comparable — enables sorting
<T extends Comparable<T>> T max(List<T> list) {
    return list.stream().max(Comparator.naturalOrder()).orElseThrow();
}

// T phải là subtype của Animal
<T extends Animal> void feed(List<T> animals) {
    animals.forEach(Animal::eat);
}
```

---

### 1.000 String Pool & Immutability

**String Pool — tối ưu bộ nhớ:**
String literals được lưu trong **String Constant Pool** (trong Heap từ Java 7). Cùng literal → cùng object reference.

```java
String a = "hello";        // literal → pool
String b = "hello";        // same pool entry
String c = new String("hello"); // new object, outside pool!

a == b        // true  — same pool reference
a == c        // false — different objects
a.equals(c)   // true  — same content

// Intern: move to pool
String d = c.intern();
a == d        // true — d now points to pool entry
```

**Tại sao String là immutable:**
1. **String Pool an toàn:** Nhiều variables chia sẻ cùng String object → nếu mutable → thay đổi ở một nơi ảnh hưởng tất cả
2. **Thread-safe by default:** Không cần synchronization khi share giữa threads
3. **HashMap key an toàn:** hashCode() không thay đổi → key luôn ở đúng bucket
4. **Security:** Filesystem paths, URLs, class names không bị mutate sau validation

**StringBuilder vs String concatenation:**
```java
// ❌ O(n²) — mỗi + tạo String mới
String s = "";
for (int i = 0; i < 10000; i++) s += i; // 10000 String objects created!

// ✅ O(n) — một buffer, append in-place
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) sb.append(i);
String result = sb.toString(); // one String at the end
```

**String methods quan trọng (Java 11+):**
```java
"  hello  ".strip()           // vs trim(): strip() handles Unicode whitespace
"hello".isBlank()             // true if empty or only whitespace
"a,b,c".split(",")           // ["a","b","c"]
String.join("-", "a","b","c") // "a-b-c"
"ab".repeat(3)               // "ababab"
"hello\nworld".lines()       // Stream<String>
"%s has %d items".formatted("Cart", 5) // "Cart has 5 items"
```

---

### 1.001 `equals()` & `hashCode()` Contract

**The Contract — phải tuân thủ:**
1. `a.equals(b)` → `a.hashCode() == b.hashCode()` (MANDATORY)
2. `a.hashCode() == b.hashCode()` → `a.equals(b)` có thể false (collision OK)
3. `equals()` phải reflexive, symmetric, transitive, consistent
4. `a.equals(null)` → phải trả `false` (không ném NPE)

**Tại sao vi phạm gây bug nghiêm trọng:**
```java
class User {
    String email;
    
    @Override
    public boolean equals(Object o) {
        User other = (User) o;
        return email.equals(other.email);
    }
    // KHÔNG override hashCode!
}

User u1 = new User("alice@test.com");
User u2 = new User("alice@test.com");
u1.equals(u2); // true — equals works

Set<User> set = new HashSet<>();
set.add(u1);
set.contains(u2); // FALSE! Different hashCode → wrong bucket → not found!
```

**`Comparable` vs `Comparator`:**
```java
// Comparable: natural ordering, baked into class
class Product implements Comparable<Product> {
    double price;
    
    @Override
    public int compareTo(Product other) {
        return Double.compare(this.price, other.price); // natural order: by price
    }
}
// Usage: Collections.sort(products); // uses compareTo

// Comparator: custom/ad-hoc ordering, external
Comparator<Product> byName = Comparator.comparing(Product::getName);
Comparator<Product> byPriceDesc = Comparator.comparingDouble(Product::getPrice).reversed();
Comparator<Product> complex = Comparator.comparing(Product::getCategory)
    .thenComparingDouble(Product::getPrice);

// Usage: products.sort(complex);
```

---

### 1.002 Annotations & Reflection

**Annotations — metadata for code:**
```java
@Retention(RetentionPolicy.RUNTIME) // visible at runtime via reflection
@Target({ElementType.METHOD, ElementType.TYPE}) // where it can be applied
public @interface Cached {
    int ttlSeconds() default 60; // element with default value
    String key() default "";
}

@Cached(ttlSeconds = 300, key = "user:{id}")
public User getUser(Long id) { ... }
```

**Retention policies:**
- `SOURCE`: chỉ trong source, bị compiler xóa (ví dụ: `@Override` — chỉ để IDE check)
- `CLASS`: trong bytecode, không visible at runtime (default)
- `RUNTIME`: visible at runtime — cần cho Spring, Hibernate, JUnit annotation processing

**Reflection — dynamic inspection:**
```java
Class<?> clazz = User.class;

// List all methods with @Cached annotation
for (Method method : clazz.getDeclaredMethods()) {
    if (method.isAnnotationPresent(Cached.class)) {
        Cached cached = method.getAnnotation(Cached.class);
        System.out.println(method.getName() + " → TTL: " + cached.ttlSeconds());
    }
}

// Invoke method dynamically
Method m = clazz.getMethod("getUser", Long.class);
User result = (User) m.invoke(serviceInstance, 42L);
```

Spring, Hibernate, JUnit tất cả đều dùng Reflection để scan annotations và wire things together. Đây là "magic" đằng sau `@Autowired`, `@Entity`, `@Test`.

---

### 1.003 Date/Time API (Java 8+)

**Tại sao `Date` và `Calendar` tệ:**
- `java.util.Date` mutable → thread-unsafe
- Month 0-indexed (January = 0) → off-by-one bugs
- No timezone handling
- `Date` thực ra chứa time, không chỉ date

**New API — tất cả immutable, clear semantics:**
```java
// Date only (no time, no timezone)
LocalDate date = LocalDate.of(2024, 3, 15); // month is 1-indexed!
LocalDate today = LocalDate.now();
LocalDate tomorrow = today.plusDays(1); // returns NEW object — immutable

// Time only
LocalTime time = LocalTime.of(14, 30, 0);

// Date + Time (no timezone)
LocalDateTime dt = LocalDateTime.of(date, time);
LocalDateTime now = LocalDateTime.now();

// Date + Time + Timezone (use for storing/comparing across zones)
ZonedDateTime zdt = ZonedDateTime.now(ZoneId.of("Asia/Ho_Chi_Minh"));
Instant instant = zdt.toInstant(); // UTC epoch millis — for DB storage

// Duration (time-based) vs Period (date-based)
Duration duration = Duration.between(startTime, endTime); // hours, minutes, seconds
Period period = Period.between(startDate, endDate);       // years, months, days

// Formatting
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm");
String formatted = now.format(fmt);                // "15/03/2024 14:30"
LocalDateTime parsed = LocalDateTime.parse("15/03/2024 14:30", fmt);
```

**When to use what:**
| Scenario | Type |
|---|---|
| Birthday, calendar event | `LocalDate` |
| Business hours | `LocalTime` |
| Log timestamp (no TZ concern) | `LocalDateTime` |
| Store in DB, compare across TZ | `Instant` |
| Show user in their timezone | `ZonedDateTime` |
| Flight departure/arrival | `ZonedDateTime` |

---

### 1.1 Record — Immutable Data Carrier

**Vấn đề trước khi có Record (Java < 16):**
Để tạo một class DTO đơn giản, bạn phải viết: fields private final, constructor, getters, `equals()`, `hashCode()`, `toString()`. Đây là **boilerplate** — code lặp lại không mang logic nghiệp vụ.

**Record giải quyết gì:**
`record` là một loại class đặc biệt mà compiler tự sinh toàn bộ boilerplate. Nó **immutable theo mặc định** — fields đều là `private final`, không có setter. Điều này bảo đảm an toàn trong môi trường concurrent vì không có shared mutable state.

```
record Point(int x, int y) {}
// Compiler sinh: constructor(x,y), x(), y(), equals, hashCode, toString
// Tương đương ~30 dòng code viết tay
```

**Compact constructor** — chạy trước khi fields được gán, dùng để validate:
```java
record Range(int min, int max) {
    Range { // compact constructor — không có parameter list riêng
        if (min > max) throw new IllegalArgumentException("min > max");
        // Sau khi constructor body kết thúc, fields được gán tự động
    }
}
```

**Khi nào dùng Record:**
- DTO (Data Transfer Object) — truyền data giữa layers
- Value Object trong Domain Driven Design
- Return type cho method trả nhiều giá trị
- Không dùng cho entity có mutable state (dùng class thường)

---

### 1.2 Sealed Class + Pattern Matching — Modeling Closed Hierarchies

**Vấn đề với inheritance thông thường:**
Khi có interface `Shape`, bất kỳ ai cũng có thể implement nó. Khi bạn viết `if (shape instanceof Circle)` bạn không biết có bao nhiêu loại Shape tồn tại — compiler không thể giúp bạn cover hết cases.

**Sealed Class giải quyết gì:**
`sealed` khai báo tường minh danh sách các subtype được phép. Compiler biết đầy đủ hierarchy → có thể check exhaustiveness trong switch.

```java
sealed interface Result<T> permits Success, Failure, Loading {}
record Success<T>(T value) implements Result<T> {}
record Failure<T>(String error) implements Result<T> {}
record Loading<T>() implements Result<T> {}
```

**Pattern Matching switch (Java 21):**
```java
// Trước: instanceof + cast thủ công
if (result instanceof Success) {
    Success s = (Success) result; // duplicate check + cast
    render(s.value());
}

// Sau: pattern matching — gọn, type-safe, compiler check exhaustive
String message = switch (result) {
    case Success<String> s -> "Got: " + s.value();
    case Failure<String> f -> "Error: " + f.error();
    case Loading<String> l -> "Loading...";
    // Không cần default vì sealed — compiler biết hết các cases
};
```

**Lợi ích thực tế:** Khi thêm case mới vào sealed hierarchy, compiler báo lỗi tại TẤT CẢ switch expressions chưa xử lý case đó. Không có case nào bị bỏ sót âm thầm.

---

### 1.3 Enum với behavior

Enum trong Java không chỉ là hằng số — mỗi instance có thể có fields và methods. Đây là **enum-as-type-safe-dictionary**.

```java
enum PaymentMethod {
    CREDIT_CARD("Credit Card", 0.03),  // 3% fee
    BANK_TRANSFER("Bank Transfer", 0.00),
    CRYPTO("Crypto", 0.01);

    final String label;
    final double feeRate;

    PaymentMethod(String label, double feeRate) {
        this.label = label;
        this.feeRate = feeRate;
    }

    double calculateFee(double amount) {
        return amount * feeRate;
    }
}
```

Thay vì `if/switch` trên string, dùng enum methods → logic gắn với data, không thể tách rời.

---

### 1.4 var — Local Variable Type Inference

`var` không phải dynamic typing. Java vẫn statically typed — compiler tự suy ra type tại compile time. Chỉ dùng được cho local variables, không dùng cho fields, parameters, return types.

```java
// ✅ Hợp lý — type rõ ràng từ right-hand side
var users = new ArrayList<User>(); // ArrayList<User>
var map = new HashMap<String, List<Integer>>(); // rõ

// ❌ Không hợp lý — type không rõ, khó đọc
var result = someService.process(); // process() trả gì?
```

---

### 1.5 Text Blocks (Java 15+)

Multiline strings không cần escape `\n`, `\"`:
```java
// Trước: khó đọc, dễ lỗi
String json = "{\n  \"name\": \"Alice\",\n  \"age\": 30\n}";

// Sau: Text Block
String json = """
    {
      "name": "Alice",
      "age": 30
    }
    """; // closing """ trên dòng riêng → trailing newline
```

---

## 2. Vấn đề thường gặp & Cách fix

### Issue 1: NullPointerException với Optional chưa nắm

```java
// ❌ Anti-pattern — tại sao sai?
// Optional.get() ném NoSuchElementException nếu empty
// Hoàn toàn đánh bất ngại lý do tồn tại của Optional
Optional<User> user = repo.findById(id);
if (user.isPresent()) {
    return user.get().getName(); // verbose và dễ quên check
}

// ✅ Idiomatic Optional
return repo.findById(id)
    .map(User::getName)
    .orElseThrow(() -> new ResourceNotFoundException("User not found: " + id));
```

### Issue 2: Enum parsing không an toàn

```java
// ❌ Ném IllegalArgumentException nếu input không match
Category.valueOf(userInput.toUpperCase()); 

// ✅ Safe parsing với custom method
public static Category fromInput(String input) {
    if (input == null) return OTHER;
    return Arrays.stream(values())
        .filter(c -> c.name().equalsIgnoreCase(input) || c.label.equalsIgnoreCase(input))
        .findFirst()
        .orElse(OTHER);
}
```

### Issue 3: Mutable Record workaround (anti-pattern)

```java
// ❌ Record có mutable collection — vi phạm immutability contract
record Config(List<String> values) {
    // values có thể bị modify từ ngoài: config.values().add("hack")
}

// ✅ Defensive copy trong compact constructor
record Config(List<String> values) {
    Config {
        values = List.copyOf(values); // immutable copy
    }
}
```

---

## 3. Code mẫu — CLI Expense Tracker

### Domain model

```java
// domain/Expense.java
package com.nguyenngoc.phase01.domain;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.UUID;

/**
 * Record là immutable — một expense không thể bị sửa sau khi tạo.
 * Muốn "sửa" → tạo expense mới với giá trị khác.
 */
public record Expense(
    String id,
    String description,
    BigDecimal amount,
    Category category,
    LocalDate date
) {
    // Compact constructor: validation chạy trước khi fields gán
    Expense {
        if (description == null || description.isBlank())
            throw new IllegalArgumentException("Description cannot be blank");
        if (amount == null || amount.compareTo(BigDecimal.ZERO) <= 0)
            throw new IllegalArgumentException("Amount must be positive");
        if (category == null)
            throw new IllegalArgumentException("Category is required");
        if (date == null)
            throw new IllegalArgumentException("Date is required");
    }

    // Static factory: đặt tên rõ ràng hơn constructor
    public static Expense create(String description, BigDecimal amount, Category category) {
        return new Expense(
            UUID.randomUUID().toString().substring(0, 8),
            description.trim(),
            amount,
            category,
            LocalDate.now()
        );
    }
}
```

```java
// domain/Category.java
package com.nguyenngoc.phase01.domain;

import java.util.Arrays;

public enum Category {
    FOOD("Ăn uống", "🍜"),
    TRANSPORT("Di chuyển", "🚗"),
    ENTERTAINMENT("Giải trí", "🎮"),
    HEALTH("Sức khỏe", "💊"),
    UTILITIES("Tiện ích", "💡"),
    OTHER("Khác", "📦");

    public final String label;
    public final String emoji;

    Category(String label, String emoji) {
        this.label = label;
        this.emoji = emoji;
    }

    /**
     * Tại sao không dùng valueOf(): valueOf không handle case-insensitive
     * và ném exception nếu không tìm thấy — không thân thiện với user input.
     */
    public static Category fromInput(String input) {
        if (input == null || input.isBlank()) return OTHER;
        return Arrays.stream(values())
            .filter(c -> c.name().equalsIgnoreCase(input)
                      || c.label.equalsIgnoreCase(input))
            .findFirst()
            .orElse(OTHER);
    }
}
```

```java
// domain/CommandResult.java — Sealed hierarchy cho command results
package com.nguyenngoc.phase01.domain;

/**
 * Sealed interface thay thế exception-based control flow cho expected outcomes.
 * Exception dành cho UNEXPECTED errors (DB down, null pointer).
 * CommandResult dành cho expected outcomes (not found, validation fail).
 */
public sealed interface CommandResult permits
    CommandResult.Success,
    CommandResult.Failure,
    CommandResult.NotFound {

    record Success(String message, Object data) implements CommandResult {
        public Success(String message) { this(message, null); }
    }

    record Failure(String reason) implements CommandResult {}

    record NotFound(String resourceType, String id) implements CommandResult {
        @Override public String toString() {
            return resourceType + " not found: " + id;
        }
    }
}
```

### Repository

```java
// repository/ExpenseRepository.java
package com.nguyenngoc.phase01.repository;

import com.nguyenngoc.phase01.domain.Expense;

import java.util.*;

public class ExpenseRepository {
    // LinkedHashMap giữ insertion order — quan trọng cho display
    private final Map<String, Expense> store = new LinkedHashMap<>();

    public Expense save(Expense expense) {
        store.put(expense.id(), expense);
        return expense;
    }

    public Optional<Expense> findById(String id) {
        return Optional.ofNullable(store.get(id));
    }

    public List<Expense> findAll() {
        return List.copyOf(store.values()); // immutable view
    }

    public boolean delete(String id) {
        return store.remove(id) != null;
    }
}
```

### Service

```java
// service/ExpenseService.java
package com.nguyenngoc.phase01.service;

import com.nguyenngoc.phase01.domain.*;
import com.nguyenngoc.phase01.repository.ExpenseRepository;

import java.math.BigDecimal;
import java.time.Month;
import java.time.YearMonth;
import java.util.*;
import java.util.stream.Collectors;

public class ExpenseService {
    private final ExpenseRepository repository;

    public ExpenseService(ExpenseRepository repository) {
        this.repository = repository;
    }

    public CommandResult add(String description, BigDecimal amount, Category category) {
        try {
            Expense expense = Expense.create(description, amount, category);
            repository.save(expense);
            return new CommandResult.Success("Added: " + expense.description(), expense);
        } catch (IllegalArgumentException e) {
            return new CommandResult.Failure(e.getMessage());
        }
    }

    public CommandResult delete(String id) {
        if (!repository.delete(id)) {
            return new CommandResult.NotFound("Expense", id);
        }
        return new CommandResult.Success("Deleted expense " + id);
    }

    /**
     * Monthly summary — thực hành Stream groupingBy + reduce
     */
    public Map<Category, BigDecimal> monthlySummary(YearMonth yearMonth) {
        return repository.findAll().stream()
            .filter(e -> YearMonth.from(e.date()).equals(yearMonth))
            .collect(Collectors.groupingBy(
                Expense::category,
                Collectors.reducing(BigDecimal.ZERO, Expense::amount, BigDecimal::add)
            ));
    }

    public String exportCsv() {
        var sb = new StringBuilder("id,description,amount,category,date\n");
        repository.findAll().forEach(e ->
            sb.append(String.join(",",
                e.id(), e.description(), e.amount().toString(),
                e.category().name(), e.date().toString()
            )).append("\n")
        );
        return sb.toString();
    }
}
```

### CLI Router

```java
// cli/ExpenseTrackerCLI.java
package com.nguyenngoc.phase01.cli;

import com.nguyenngoc.phase01.domain.*;
import com.nguyenngoc.phase01.service.ExpenseService;

import java.math.BigDecimal;
import java.util.Scanner;

public class ExpenseTrackerCLI {
    private final ExpenseService service;
    private final Scanner scanner;

    public ExpenseTrackerCLI(ExpenseService service) {
        this.service = service;
        this.scanner = new Scanner(System.in);
    }

    public void start() {
        System.out.println("=== Expense Tracker ===");
        System.out.println("Commands: add, list, delete, summary, export, quit");

        while (true) {
            System.out.print("\n> ");
            String[] parts = scanner.nextLine().trim().split("\\s+", 4);
            String command = parts[0].toLowerCase();

            // Pattern matching switch — Java 21
            // Sealed hierarchy: compiler sẽ báo lỗi nếu có case chưa xử lý
            CommandResult result = switch (command) {
                case "add" -> handleAdd(parts);
                case "list" -> handleList();
                case "delete" -> parts.length > 1
                    ? service.delete(parts[1])
                    : new CommandResult.Failure("Usage: delete <id>");
                case "summary" -> handleSummary();
                case "export" -> handleExport();
                case "quit", "exit" -> { System.out.println("Bye!"); yield null; }
                default -> new CommandResult.Failure("Unknown command: " + command);
            };

            if (result == null) break;
            handleResult(result);
        }
    }

    private CommandResult handleAdd(String[] parts) {
        if (parts.length < 3) return new CommandResult.Failure("Usage: add <amount> <desc> [category]");
        try {
            BigDecimal amount = new BigDecimal(parts[1]);
            String description = parts[2];
            Category category = parts.length > 3 ? Category.fromInput(parts[3]) : Category.OTHER;
            return service.add(description, amount, category);
        } catch (NumberFormatException e) {
            return new CommandResult.Failure("Invalid amount: " + parts[1]);
        }
    }

    private CommandResult handleList() {
        // pattern matching instanceof — tất nhiên mọi thứ ổn nếu đây là list
        return new CommandResult.Success("Listed", null);
    }

    private CommandResult handleSummary() {
        var yearMonth = java.time.YearMonth.now();
        var summary = service.monthlySummary(yearMonth);
        summary.forEach((cat, total) ->
            System.out.printf("  %s %-15s %,.0f VND%n", cat.emoji, cat.label, total)
        );
        return new CommandResult.Success("Summary for " + yearMonth);
    }

    private CommandResult handleExport() {
        System.out.println(service.exportCsv());
        return new CommandResult.Success("Exported to console");
    }

    /**
     * Xử lý kết quả với pattern matching — exhaustive, type-safe.
     * Nếu thêm case mới vào CommandResult sealed, compiler báo lỗi ngay tại đây.
     */
    private void handleResult(CommandResult result) {
        switch (result) {
            case CommandResult.Success s -> System.out.println("✅ " + s.message());
            case CommandResult.Failure f -> System.out.println("❌ " + f.reason());
            case CommandResult.NotFound nf -> System.out.println("🔍 " + nf);
        }
    }
}
```

---

## 4. Lab Steps

1. **Setup:** Tạo package structure theo tree trên, thêm dependency cho maven nếu cần
2. **Record validation:** Thử tạo `Expense.create("", new BigDecimal(-1), null)` → xem exception message
3. **Enum:** In tất cả categories với label và emoji: `Arrays.stream(Category.values()).forEach(System.out::println)`
4. **Sealed switch:** Thêm một case mới vào `CommandResult` (ví dụ `Partial`) → observe compiler error trong `handleResult`
5. **Stream:** Trong `monthlySummary`, thêm `peek(e -> System.out.println("Processing: " + e.id()))` để xem lazy evaluation
6. **Text block:** Tạo JSON export: `String json = """{"id":"%s","amount":%s}""".formatted(e.id(), e.amount())`
7. **var:** Refactor một method dùng `var` — xem IDE vẫn show type khi hover

---

## 5. Checklist tự kiểm tra

- [ ] Record compact constructor ném exception cho invalid data
- [ ] `Category.fromInput("FOOD")` và `Category.fromInput("food")` và `Category.fromInput("Ăn uống")` đều ra `Category.FOOD`
- [ ] Switch expression với sealed interface không cần `default` branch
- [ ] Pattern matching: `case Success s ->` không cần cast `(Success) result`
- [ ] `Expense.date()` trả `LocalDate`, không phải `getDate()` — record tự sinh accessor với tên field
- [ ] `List.copyOf(store.values())` trong repository → caller không thể modify internal state
