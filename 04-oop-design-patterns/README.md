# Phase 04 — OOP & Design Patterns

> **Mục tiêu:** Biết thiết kế code thay vì chỉ làm code chạy. Hiểu tại sao pattern tồn tại, không học thuộc code.

---

## Part 1 — OOP Principles

### 4 Pillars
| Pillar | Ý nghĩa | Sai lầm thường gặp |
|--------|---------|-------------------|
| **Encapsulation** | Ẩn implementation, expose interface | Getter/setter cho mọi field không phải encapsulation |
| **Abstraction** | Expose what, hide how | Abstraction không phải interface = abstract class |
| **Inheritance** | Reuse, is-a relationship | Dùng inheritance để reuse code khi không có is-a → fragile base class |
| **Polymorphism** | One interface, many forms | Method overloading ≠ polymorphism; overriding mới là |

### Composition vs Inheritance

```
Inheritance: "is-a"    → Car is-a Vehicle
Composition: "has-a"   → Car has-a Engine
```

**Prefer composition over inheritance** — tại sao:
- Inheritance là compile-time coupling (rigid)
- Composition là runtime coupling (flexible)
- Fragile base class problem: change parent → break child

---

## Part 2 — SOLID

| Principle | Tên đầy đủ | Một câu |
|-----------|-----------|---------|
| **S** | Single Responsibility | Class chỉ có một lý do để thay đổi |
| **O** | Open/Closed | Open for extension, closed for modification |
| **L** | Liskov Substitution | Subtype phải thay thế được supertype |
| **I** | Interface Segregation | Client không nên phụ thuộc interface nó không dùng |
| **D** | Dependency Inversion | High-level module phụ thuộc abstraction, không phải concrete |

### Liskov Substitution — vi phạm phổ biến
```java
// Vi phạm LSP
class Rectangle { setWidth(w); setHeight(h); }
class Square extends Rectangle {
    @Override setWidth(w) { this.width = w; this.height = w; } // ← vi phạm!
}

// Nếu code dùng Rectangle expect setWidth không ảnh hưởng height
// → Square phá vỡ kỳ vọng → không thể substitute
```

---

## Part 3 — Design Principles

| Principle | Mô tả |
|-----------|-------|
| **DRY** | Don't Repeat Yourself — mỗi piece of knowledge có một nơi authoritative |
| **KISS** | Keep It Simple, Stupid — solution đơn giản nhất đủ giải quyết bài toán |
| **YAGNI** | You Aren't Gonna Need It — không thêm feature chưa cần |
| **Cohesion** | Các thứ trong một module liên quan chặt chẽ với nhau |
| **Coupling** | Mức độ phụ thuộc giữa các module — low coupling = dễ thay đổi |

---

## Part 4 — Design Patterns (12 pattern trọng tâm)

### Creational

#### Strategy
```
Định nghĩa family of algorithms, encapsulate từng cái, làm chúng interchangeable
```
```java
interface PaymentStrategy { void pay(int amount); }
class CreditCard implements PaymentStrategy { ... }
class PayPal implements PaymentStrategy { ... }

class PaymentProcessor {
    private PaymentStrategy strategy;  // inject at runtime
    void process(int amount) { strategy.pay(amount); }
}
```
**Dùng khi:** Có nhiều variant của cùng một behavior; muốn swap algorithm lúc runtime.

#### Factory Method
```
Define interface for creating object, let subclass decide which class to instantiate
```
```java
abstract class NotificationFactory {
    abstract Notification create(String type);
}
```
**Dùng khi:** Không biết trước class cụ thể; muốn subclass quyết định.

#### Builder
```
Construct complex object step by step
```
```java
Order order = Order.builder()
    .customerId(id)
    .item("SKU-001", 2)
    .item("SKU-002", 1)
    .discount(10)
    .build();
```
**Dùng khi:** Constructor có quá nhiều parameters; object cần built theo nhiều bước optional.

### Structural

#### Adapter
```
Convert interface của một class thành interface khác mà client expect
```
```java
// Legacy system trả về XML, service mới cần JSON
class XmlToJsonAdapter implements JsonDataProvider {
    private LegacyXmlProvider legacy;
    public JsonData getData() { return convert(legacy.getXml()); }
}
```

#### Decorator
```
Thêm behavior vào object mà không thay đổi class
```
```java
InputStream raw    = new FileInputStream("file.txt");
InputStream buf    = new BufferedInputStream(raw);    // decorator
InputStream cipher = new CipherInputStream(buf, key); // decorator
```
**Dùng khi:** Cần add feature mà không dùng inheritance; cần stack nhiều feature.

#### Proxy
```
Provide surrogate/placeholder cho object khác — control access
```
- **Virtual Proxy:** Lazy loading (Hibernate lazy entity)
- **Protection Proxy:** Access control (Spring Security)
- **Remote Proxy:** RPC (RMI, gRPC stub)
- **Caching Proxy:** Cache results

#### Facade
```
Provide simple interface cho complex subsystem
```
**Dùng khi:** Subsystem phức tạp; muốn cung cấp high-level API đơn giản.

### Behavioral

#### Template Method
```
Define skeleton of algorithm trong base class, defer steps cụ thể xuống subclass
```
```java
abstract class DataProcessor {
    final void process() {     // ← template method
        readData();
        processData();         // ← abstract, subclass implement
        writeData();
    }
    abstract void processData();
}
```

#### Observer
```
Define one-to-many dependency — khi subject thay đổi, tất cả observer được notify
```
```java
// Java event model, Spring ApplicationEvent là Observer pattern
eventPublisher.publishEvent(new OrderCreatedEvent(order));
```

#### Command
```
Encapsulate request as object — support undo, queue, logging
```
```java
interface Command { void execute(); void undo(); }
class TransferCommand implements Command {
    void execute() { account.debit(amount); }
    void undo()    { account.credit(amount); }
}
```

#### Chain of Responsibility
```
Pass request along chain of handlers — mỗi handler decide xử lý hay pass tiếp
```
**Ví dụ thực tế:** Spring `SecurityFilterChain`, Servlet `FilterChain`, logging handlers.

#### State
```
Object thay đổi behavior khi internal state thay đổi
```
```java
// Order: PENDING → CONFIRMED → SHIPPED → DELIVERED
// Mỗi state define hành vi cho các action khác nhau
interface OrderState {
    void confirm(Order order);
    void ship(Order order);
    void deliver(Order order);
}
```

---

## Project — Banking / Payment Engine

```
src/
├── model/
│   ├── Account.java
│   ├── Transaction.java      (Record)
│   └── TransactionType.java  (Enum)
├── payment/
│   ├── PaymentStrategy.java          (Strategy pattern)
│   ├── CreditCardPayment.java
│   ├── BankTransferPayment.java
│   └── PaymentProcessor.java
├── notification/
│   ├── NotificationService.java      (Observer pattern)
│   ├── EmailNotifier.java
│   └── SmsNotifier.java
├── transaction/
│   ├── TransactionCommand.java       (Command pattern)
│   ├── TransferCommand.java
│   └── TransactionHistory.java       (undo stack)
├── account/
│   ├── AccountState.java             (State pattern)
│   ├── ActiveState.java
│   ├── FrozenState.java
│   └── ClosedState.java
├── factory/
│   └── AccountFactory.java           (Factory Method)
└── Main.java
```

**Yêu cầu:**
1. Transfer money giữa accounts — Command pattern + undo
2. Multiple payment methods — Strategy pattern
3. Account state machine: Active → Frozen → Closed — State pattern
4. Notify on transaction — Observer pattern
5. Không có Spring, không có database — in-memory

---

## Checklist — 6 câu hỏi

Ví dụ với `Strategy Pattern`:

| # | Câu hỏi | Trả lời |
|---|---------|---------|
| 1 | Là gì? | Encapsulate interchangeable algorithms behind common interface |
| 2 | Giải quyết gì? | Tránh if-else/switch khi có nhiều variant của cùng behavior |
| 3 | Hoạt động thế nào? | Context holds reference to Strategy, delegates to it at runtime |
| 4 | Khi nào dùng? | Payment methods, sorting algorithms, compression formats |
| 5 | Khi nào không? | Khi chỉ có 1-2 variant cố định — over-engineering |
| 6 | Debug thế nào? | Trace which strategy was injected; kiểm tra strategy thay đổi có đúng không |

---

## Run

```bash
mvn test -pl phase-04-oop-design-patterns
```
