# Phase 01 — Java Language

> **Mục tiêu:** Thành thạo Java language ở mức đọc và viết code tự tin, hiểu từng keyword có ý nghĩa gì và tại sao nó tồn tại.

---

## Topics

### 1. Variables & Types
| Concept | Nội dung |
|---------|---------|
| Primitive | `byte` `short` `int` `long` `float` `double` `char` `boolean` — kích thước, range, default value |
| Reference | Object, array — stored on heap, variable holds reference |
| Autoboxing | `int` ↔ `Integer` — khi nào boxing xảy ra, chi phí là gì |
| Type casting | Widening (implicit) vs Narrowing (explicit), integer overflow |
| `var` | Local variable type inference (Java 10+), giới hạn sử dụng |

### 2. Class & Object
| Concept | Nội dung |
|---------|---------|
| Class | Fields, methods, constructors, this |
| Object | `new`, heap allocation, reference vs value |
| Access Modifier | `public` `protected` `package-private` `private` — quy tắc tầm vực |
| `static` | Static field vs instance field, static method, static block |
| `final` | Final variable (constant), final method (no override), final class (no extend) |
| Nested class | Static nested class vs inner class — khi nào dùng cái nào |
| Anonymous class | Inline implementation — liên quan đến functional interface |

### 3. Interface & Abstract Class
| Concept | Nội dung |
|---------|---------|
| Interface | `default`, `static`, `private` method (Java 8/9+) |
| Abstract class | Partial implementation, constructor, fields |
| Khi nào dùng gì | Interface = contract / capability; Abstract = shared implementation |
| Functional Interface | `@FunctionalInterface`, liên quan đến lambda |

### 4. Enum
| Concept | Nội dung |
|---------|---------|
| Enum basics | Singleton by design, `.values()`, `.ordinal()`, `.name()` |
| Enum với fields | Constructor, custom methods |
| Enum với abstract | Switch-like polymorphism |
| EnumMap / EnumSet | Hiệu năng so với HashMap/HashSet |

### 5. Record (Java 16+)
| Concept | Nội dung |
|---------|---------|
| Record | Immutable data carrier — tự động có constructor, accessor, equals/hashCode/toString |
| Compact constructor | Validation trong record |
| Khi nào dùng | DTO, value object, response body |

### 6. Sealed Class (Java 17+)
| Concept | Nội dung |
|---------|---------|
| `sealed` / `permits` | Giới hạn subclass — exhaustive pattern matching |
| `non-sealed` | Mở lại hierarchy |
| Kết hợp `switch` | Sealed + pattern matching = algebraic data types kiểu Java |

### 7. Pattern Matching (Java 16–21)
| Concept | Nội dung |
|---------|---------|
| `instanceof` pattern | `if (obj instanceof String s)` — không cần cast |
| Switch expression | `yield`, arrow label |
| Pattern switch | `case String s when s.isEmpty()` |
| Guarded pattern | `when` clause |

### 8. Generics
| Concept | Nội dung |
|---------|---------|
| Generic class/method | `<T>`, `<T extends Comparable<T>>` |
| Wildcard | `?`, `? extends T` (upper bounded), `? super T` (lower bounded) |
| Type Erasure | Tại sao `List<String>` và `List<Integer>` là cùng 1 class tại runtime |
| PECS | Producer Extends, Consumer Super |

### 9. String
| Concept | Nội dung |
|---------|---------|
| String Pool | String literal → pool, `new String()` → heap |
| Immutability | Tại sao String immutable, hệ quả với `+` concatenation |
| StringBuilder | Mutable, `append()`, khi nào dùng thay String |
| String API | `split`, `strip`, `isBlank`, `repeat`, `formatted` (Java 15+) |
| Text Block | Multi-line string (Java 15+) |

### 10. Object Methods
| Concept | Nội dung |
|---------|---------|
| `equals()` | Contract: reflexive, symmetric, transitive, consistent, null-safe |
| `hashCode()` | Contract với equals, hash collision, load factor |
| `toString()` | Debugging, logging |
| `Comparable` vs `Comparator` | Natural ordering vs custom ordering |
| `clone()` | Shallow vs deep copy — tại sao ít dùng |

### 11. Annotations & Reflection
| Concept | Nội dung |
|---------|---------|
| Built-in annotations | `@Override`, `@Deprecated`, `@SuppressWarnings`, `@FunctionalInterface` |
| Custom annotation | `@Retention`, `@Target`, `@Inherited` |
| Reflection | `Class`, `Method`, `Field` — đọc và gọi tại runtime |
| Reflection dùng ở đâu | Spring IoC, Jackson, JUnit — đây là lý do phải hiểu |

### 12. Date/Time API (Java 8+)
| Concept | Nội dung |
|---------|---------|
| `LocalDate`, `LocalTime`, `LocalDateTime` | Không có timezone |
| `ZonedDateTime`, `Instant` | Với timezone / epoch |
| `Duration`, `Period` | Khoảng thời gian |
| `DateTimeFormatter` | Format / parse |
| Tại sao không dùng `Date`/`Calendar` | Thread-safe, immutable, API tốt hơn |

### 13. Regex
| Concept | Nội dung |
|---------|---------|
| `Pattern`, `Matcher` | Compile once, match nhiều lần |
| Groups, quantifiers | `(.+)`, `\d{3}`, `(?:...)` |
| Khi nào dùng | Validation, parsing, extraction |

---

## Project — CLI Expense Tracker

**Không dùng Spring, không dùng database. Chỉ Java thuần.**

```
expense-tracker/
├── model/
│   ├── Expense.java          (Record)
│   ├── Category.java         (Enum)
│   └── ExpenseSummary.java   (Record)
├── repository/
│   └── ExpenseRepository.java
├── service/
│   └── ExpenseService.java
├── cli/
│   └── ExpenseTrackerCLI.java
└── Main.java
```

**Features:**
- Thêm / xoá / sửa expense
- Lọc theo category, date range
- Tổng hợp theo tháng
- Export CSV (dùng `Files`, `BufferedWriter`)
- Persist bằng JSON file (dùng `ObjectMapper` hoặc tự serialize)

**Java features cần dùng:**
- Record cho `Expense`, `ExpenseSummary`
- Enum cho `Category`
- Sealed class cho command result (`Success`, `Failure`, `NotFound`)
- Pattern matching switch cho command routing
- `LocalDate`, `DateTimeFormatter`
- `Optional` để tránh null

---

## Checklist — 6 câu hỏi cho mỗi concept

Ví dụ với `String Pool`:

| # | Câu hỏi | Trả lời mẫu |
|---|---------|-------------|
| 1 | Nó là gì? | Vùng bộ nhớ đặc biệt trong heap lưu String literal |
| 2 | Giải quyết gì? | Tái sử dụng String giống nhau, tiết kiệm bộ nhớ |
| 3 | Hoạt động thế nào? | `"hello"` → tìm trong pool, nếu có thì trả ref, nếu không thì tạo mới |
| 4 | Khi nào dùng? | Luôn dùng String literal khi có thể |
| 5 | Khi nào không? | `new String("hello")` bypass pool — không dùng trừ khi cố ý |
| 6 | Debug thế nào? | `==` vs `.equals()` fail → khả năng cao là pool vs heap confusion |

---

## Run

```bash
mvn test -pl phase-01-java-language
mvn compile -pl phase-01-java-language
```
