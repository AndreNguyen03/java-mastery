# Phase 04 — OOP & Design Patterns · Lý thuyết & Lab

---

## 1. Lý thuyết

### 1.0 Bốn Trụ Cột OOP

**Encapsulation — đóng gói:**
Che giấu internal state, chỉ expose qua public API. Lý do: internal representation có thể thay đổi (ArrayList → LinkedList) mà caller code không biết.

```java
// ❌ Không encapsulate — expose internal state
class BankAccount {
    public double balance; // anyone can set to negative!
}

// ✅ Encapsulate — control access
class BankAccount {
    private double balance; // internal detail

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        balance += amount;
    }

    public double getBalance() { return balance; } // read-only access
}
```

**Abstraction — trừu tượng hóa:**
Ẩn complexity, chỉ expose what's relevant. `List` interface ẩn đi ArrayLst vs LinkedList implementation. Caller chỉ cần biết `add()`, `get()`, không cần biết cách store.

**Inheritance — kế thừa:**
Tái sử dụng code từ parent class. Nhưng: **"prefer composition over inheritance"** — inheritance tạo tight coupling, hard to change parent without breaking children.

```java
// ❌ Inheritance khi không thực sự là "is-a"
class Stack<T> extends ArrayList<T> { // Stack IS-NOT-A ArrayList
    // Inherits add(index, elem), remove(index), etc. — violates stack contract!
}

// ✅ Composition — Stack HAS-A list
class Stack<T> {
    private final Deque<T> storage = new ArrayDeque<>();
    public void push(T item) { storage.push(item); }
    public T pop() { return storage.pop(); }
    // Only stack operations exposed — correct abstraction
}
```

**Polymorphism — đa hình:**
Cùng interface, nhiều implementations. Runtime method dispatch:

```java
List<Shape> shapes = List.of(new Circle(5), new Rectangle(3, 4), new Triangle(3, 4, 5));
// Each shape.area() calls the correct override — runtime dispatch
double totalArea = shapes.stream().mapToDouble(Shape::area).sum();
```

**Composition vs Inheritance — khi nào dùng gì:**
- Inheritance: "is-a" relationship là thực sự đúng AND muốn share implementation
- Composition: "has-a" hoặc khi cần flexibility (swap implementation), tránh deep hierarchies

---

### 1.00 Factory Method & Abstract Factory

**Factory Method — tạo object mà không hard-code class:**

**Vấn đề:** `new PaymentProcessor("CARD")` → hard-coded. Muốn thêm loại mới → phải sửa caller.

```java
// Factory Method Pattern
interface PaymentProcessor {
    boolean process(double amount);
}

class PaymentProcessorFactory {
    public static PaymentProcessor create(PaymentType type) {
        return switch (type) {
            case CREDIT_CARD -> new CreditCardProcessor();
            case BANK_TRANSFER -> new BankTransferProcessor();
            case CRYPTO -> new CryptoProcessor();
        };
    }
}

// Caller không biết concrete class
PaymentProcessor processor = PaymentProcessorFactory.create(PaymentType.CREDIT_CARD);
processor.process(100000);
// Thêm PAYPAL: chỉ thêm case trong factory + class PayPalProcessor
// → Caller code không thay đổi — Open/Closed Principle!
```

**Abstract Factory — factory của factories:**
Tạo families of related objects mà không specify concrete classes.

```java
// Tạo UI components cho nhiều platforms (Dark/Light theme)
interface UIFactory {
    Button createButton();
    TextField createTextField();
    Dialog createDialog();
}

class DarkThemeFactory implements UIFactory {
    public Button createButton() { return new DarkButton(); }
    public TextField createTextField() { return new DarkTextField(); }
    public Dialog createDialog() { return new DarkDialog(); }
}

class LightThemeFactory implements UIFactory {
    public Button createButton() { return new LightButton(); }
    // ...
}

// Application chỉ biết UIFactory interface
class App {
    private final UIFactory uiFactory;
    public App(UIFactory factory) { this.uiFactory = factory; }
    // Switch theme = swap factory = all components change consistently
}
```

---

### 1.000 Template Method Pattern

**Vấn đề:** Nhiều algorithms có cùng skeleton nhưng khác nhau ở một số steps. Copy-paste skeleton → duplicate code, khó maintain.

**Giải pháp:** Abstract class định nghĩa algorithm skeleton với `final` method. Subclass override specific steps.

```java
// Template Method: algorithm skeleton in abstract class
abstract class ReportGenerator {

    // Template method — defines the algorithm, final to prevent override
    public final void generate(String filename) {
        List<String> data = fetchData();        // step 1
        List<String> processed = process(data); // step 2
        String formatted = format(processed);   // step 3 — abstract: subclass implements
        saveToFile(formatted, filename);        // step 4
    }

    // Hook — optional override with default behavior
    protected List<String> fetchData() {
        return loadFromDatabase(); // default: DB
    }

    // Abstract steps — MUST be implemented by subclasses
    protected abstract List<String> process(List<String> data);
    protected abstract String format(List<String> processed);

    private void saveToFile(String content, String filename) {
        // ... common save logic
    }
}

class CsvReportGenerator extends ReportGenerator {
    protected List<String> process(List<String> data) {
        return data.stream().map(String::toUpperCase).toList();
    }
    protected String format(List<String> processed) {
        return String.join(",", processed); // CSV format
    }
}

class HtmlReportGenerator extends ReportGenerator {
    @Override
    protected List<String> fetchData() {
        return loadFromApi(); // override hook — different data source
    }
    protected List<String> process(List<String> data) { /* ... */ return data; }
    protected String format(List<String> processed) {
        return "<html><body>" + String.join("<br>", processed) + "</body></html>";
    }
}
```

**Hollywood Principle:** "Don't call us, we'll call you." Abstract class gọi abstract methods của subclass — not the other way.

---

### 1.0000 Adapter Pattern

**Vấn đề:** Có interface A, cần dùng với code expect interface B. Không thể sửa A (third-party) hoặc B (legacy).

```java
// Legacy code expects: LegacyPrinter
interface LegacyPrinter {
    void printDocument(String content);
}

// New library provides: ModernPdfPrinter
class ModernPdfPrinter {
    public void renderAndPrint(byte[] pdfBytes) { /* ... */ }
}

// Adapter: wraps ModernPdfPrinter, exposes LegacyPrinter interface
class PdfPrinterAdapter implements LegacyPrinter {
    private final ModernPdfPrinter modernPrinter;

    PdfPrinterAdapter(ModernPdfPrinter modernPrinter) {
        this.modernPrinter = modernPrinter;
    }

    @Override
    public void printDocument(String content) {
        byte[] pdfBytes = convertToPdf(content); // adaptation
        modernPrinter.renderAndPrint(pdfBytes);
    }
}

// Existing code works unchanged
LegacyPrinter printer = new PdfPrinterAdapter(new ModernPdfPrinter());
printer.printDocument("Hello World"); // works!
```

---

### 1.00000 Chain of Responsibility

**Vấn đề:** Một request cần đi qua nhiều handlers. Không biết trước handler nào sẽ xử lý. Muốn thêm/bớt handlers mà không sửa sender.

```java
abstract class RequestHandler {
    private RequestHandler next;

    public RequestHandler setNext(RequestHandler next) {
        this.next = next;
        return next; // enables chaining: h1.setNext(h2).setNext(h3)
    }

    public void handle(Request request) {
        if (canHandle(request)) {
            process(request);
        } else if (next != null) {
            next.handle(request); // pass to next in chain
        }
    }

    protected abstract boolean canHandle(Request request);
    protected abstract void process(Request request);
}

class AuthHandler extends RequestHandler {
    protected boolean canHandle(Request r) { return !r.isAuthenticated(); }
    protected void process(Request r) { throw new UnauthorizedException(); }
}

class RateLimitHandler extends RequestHandler {
    protected boolean canHandle(Request r) { return isRateLimited(r.getClientIp()); }
    protected void process(Request r) { throw new TooManyRequestsException(); }
}

class BusinessLogicHandler extends RequestHandler {
    protected boolean canHandle(Request r) { return true; } // default handler
    protected void process(Request r) { processBusinessLogic(r); }
}

// Setup chain
RequestHandler chain = new AuthHandler();
chain.setNext(new RateLimitHandler()).setNext(new BusinessLogicHandler());
chain.handle(request);
// Chain: Auth check → Rate limit check → Business logic
```

**Spring Security Filter Chain** là Chain of Responsibility pattern — mỗi security filter là một handler.

---

### 1.1 SOLID Principles — Tại sao cần

**Single Responsibility (SRP):** Một class chỉ có một lý do để thay đổi. Nếu `UserService` vừa xử lý logic nghiệp vụ vừa gửi email vừa log — khi email server đổi, bạn phải sửa UserService. Tách ra `EmailService` riêng.

**Open/Closed (OCP):** Mở để mở rộng, đóng để sửa đổi. Thay vì `if (paymentType == "CREDIT_CARD") ... else if (paymentType == "BANK")...`, dùng Strategy pattern — thêm payment method mới mà không sửa code cũ.

**Liskov Substitution (LSP):** Subtype phải thay thế được supertype mà không phá vỡ behavior. Vi phạm kinh điển: `Square extends Rectangle` — `setWidth` trên Square cũng phải đổi height → unexpected behavior khi code nhận Rectangle nhưng nhận Square.

**Interface Segregation (ISP):** Đừng force client implement method họ không dùng. `Flyable` và `Swimmable` riêng thay vì `FlyingSwimmingAnimal` chung.

**Dependency Inversion (DIP):** Depend on abstractions, not concretions. `OrderService(EmailService)` → hard to test. `OrderService(NotificationService)` với `EmailService implements NotificationService` → dễ test với mock.

---

### 1.2 Design Patterns

**Strategy Pattern — "Behavior as Object"**

**Vấn đề:** Có nhiều thuật toán/cách làm, cần switch runtime.
**Giải pháp:** Định nghĩa interface cho behavior, implement nhiều strategies, inject vào context.

```
Trước: if (type == CARD) { ...50 lines... } else if (type == BANK) { ...60 lines... }
Sau: strategy.pay(amount) — context không biết strategy cụ thể là gì
```

**Lợi ích:**
- Thêm payment method mới: chỉ thêm class mới, không sửa processor
- Test: inject mock strategy
- Runtime switch: `processor.setStrategy(new CryptoPayment())`

---

**Command Pattern — "Action as Object" + Undo**

**Vấn đề:** Cần: (1) decouple sender từ receiver, (2) queue/log commands, (3) undo/redo.
**Giải pháp:** Đóng gói một action (và state cần để undo nó) thành object.

```java
interface Command {
    void execute();
    void undo();
}
```

**Lợi ích:**
- History: stack of commands → undo by popping
- Replay: execute command list lại từ đầu
- Logging: serialize commands → audit trail

---

**State Pattern — "Context behavior changes based on state"**

**Vấn đề:** Object có behavior khác nhau tùy state hiện tại. `if (state == ACTIVE) {...} else if (state == FROZEN) {...}` — mỗi method phải kiểm tra state → code khó maintain khi thêm states.

**Giải pháp:** Mỗi state là một object riêng chứa behavior của state đó. Context delegate cho current state.

```
Account(activeState) → account.withdraw() → activeState.withdraw()
Account(frozenState) → account.withdraw() → frozenState.withdraw() → throws exception
```

**Transition logic trong state:** `activeState.freeze()` → `context.setState(frozenState)` — state tự biết transition rules.

---

**Observer Pattern — "Event-driven communication"**

**Vấn đề:** Khi một object thay đổi, nhiều objects khác cần biết, nhưng không muốn coupling chặt.

**Giải pháp:** Subject maintains danh sách observers, notify tất cả khi state đổi.

**Publisher-Subscriber vs Observer:**
Observer: subscriber biết publisher trực tiếp.
Pub/Sub: có event bus ở giữa — publisher và subscriber không biết nhau (loose coupling hơn, phức tạp hơn).

---

**Builder Pattern — "Step-by-step construction"**

**Vấn đề:** Constructor với nhiều optional parameters → `new User(name, null, null, email, null, true)` — confusing, error-prone.

**Giải pháp:** Fluent builder với named methods, validation trước khi build.

**Telescoping constructor anti-pattern:**
```java
User(String name) { this(name, null); }
User(String name, String email) { this(name, email, null); }
User(String name, String email, String phone) { ... } // grows indefinitely
```

---

**Decorator Pattern — "Add behavior without subclassing"**

**Vấn đề:** Muốn thêm features vào object (logging, caching, retry) mà không subclass vì:
- Subclassing → combinatorial explosion (LoggingRetryingCachingService)
- Có thể không biết concrete class lúc runtime

**Giải pháp:** Wrapper implements same interface, delegates to wrapped object, adds behavior.

```java
Service base = new UserService();
Service withLogging = new LoggingDecorator(base);
Service withCaching = new CachingDecorator(withLogging);
// withCaching.get() → check cache → withLogging.get() → log → base.get()
```

---

**Proxy Pattern — "Control access to object"**

Giống Decorator nhưng mục đích khác:
- Decorator: add behavior
- Proxy: control access (lazy init, auth check, remote call)

Spring AOP dùng proxy để implement `@Transactional`, `@Cacheable` mà không sửa code service.

---

### 1.3 Tại sao DRY/KISS/YAGNI quan trọng

**DRY (Don't Repeat Yourself):** Code trùng lặp là technical debt. Khi logic đổi, phải sửa ở nhiều nơi — dễ quên → bugs. Extract method, base class, utility.

**KISS (Keep It Simple):** Đừng over-engineer. Simple code dễ đọc, debug, maintain hơn. "Premature optimization is the root of all evil" (Knuth).

**YAGNI (You Aren't Gonna Need It):** Đừng implement feature bạn nghĩ sẽ cần. 90% features "có thể cần" không bao giờ được dùng nhưng làm code phức tạp.

---

## 2. Vấn đề thường gặp & Cách fix

### Issue 1: Over-engineering với patterns

```java
// ❌ Dùng Factory cho một loại object duy nhất — không cần thiết
class UserFactory {
    static User create(String name) { return new User(name); }
}

// ✅ Chỉ dùng Factory khi có nhiều implementations cần chọn runtime
class PaymentProcessorFactory {
    static PaymentProcessor create(PaymentType type) {
        return switch (type) {
            case CARD -> new CardProcessor();
            case BANK -> new BankProcessor();
            case CRYPTO -> new CryptoProcessor();
        };
    }
}
```

### Issue 2: Builder không validate

```java
// ❌ Build succeeds nhưng object invalid
User user = new User.Builder().name("Alice").build(); // email missing!

// ✅ Validate trong build()
public User build() {
    Objects.requireNonNull(name, "name is required");
    if (email == null || !email.contains("@"))
        throw new IllegalStateException("Valid email required");
    return new User(this);
}
```

### Issue 3: State transitions không hợp lệ

```java
// ❌ State transition không được guard
account.close(); // account đã CLOSED → close lại?

// ✅ Guard trong state
class ClosedState implements AccountState {
    void close(Account account) {
        throw new IllegalStateException("Account already closed");
    }
    void deposit(Account account, double amount) {
        throw new IllegalStateException("Cannot deposit to closed account");
    }
}
```

---

## 3. Code mẫu — Banking System

```java
// patterns/strategy/PaymentStrategy.java
public interface PaymentStrategy {
    boolean pay(double amount, String description);
    String getName();
}

// patterns/strategy/CreditCardPayment.java
public class CreditCardPayment implements PaymentStrategy {
    private final String cardNumber;
    private double availableLimit;

    public CreditCardPayment(String cardNumber, double limit) {
        this.cardNumber = cardNumber;
        this.availableLimit = limit;
    }

    @Override
    public boolean pay(double amount, String description) {
        if (amount > availableLimit) {
            System.out.printf("❌ Insufficient credit limit (%.0f < %.0f)%n", availableLimit, amount);
            return false;
        }
        availableLimit -= amount;
        System.out.printf("✅ Credit card payment: %.0f for '%s' (remaining limit: %.0f)%n",
            amount, description, availableLimit);
        return true;
    }

    @Override public String getName() { return "Credit Card *" + cardNumber.substring(cardNumber.length()-4); }
}

// patterns/strategy/PaymentProcessor.java
public class PaymentProcessor {
    private PaymentStrategy strategy;

    public PaymentProcessor(PaymentStrategy strategy) {
        this.strategy = strategy;
    }

    public void setStrategy(PaymentStrategy strategy) {
        this.strategy = strategy;
    }

    public boolean processPayment(double amount, String description) {
        System.out.println("Processing with: " + strategy.getName());
        return strategy.pay(amount, description);
    }
}
```

```java
// patterns/command/Command.java
public interface Command {
    void execute();
    void undo();
    String getDescription();
}

// patterns/command/TransferCommand.java
public class TransferCommand implements Command {
    private final BankAccount from;
    private final BankAccount to;
    private final double amount;
    private boolean executed = false;

    public TransferCommand(BankAccount from, BankAccount to, double amount) {
        this.from = from;
        this.to = to;
        this.amount = amount;
    }

    @Override
    public void execute() {
        if (executed) throw new IllegalStateException("Command already executed");
        from.withdraw(amount);
        to.deposit(amount);
        executed = true;
    }

    @Override
    public void undo() {
        if (!executed) throw new IllegalStateException("Command not yet executed");
        to.withdraw(amount);
        from.deposit(amount);
        executed = false;
    }

    @Override public String getDescription() {
        return String.format("Transfer %.0f from %s to %s", amount, from.getId(), to.getId());
    }
}

// patterns/command/TransactionHistory.java
public class TransactionHistory {
    private final Deque<Command> history = new ArrayDeque<>();
    private final int maxSize;

    public TransactionHistory(int maxSize) { this.maxSize = maxSize; }

    public void execute(Command command) {
        command.execute();
        history.push(command);
        if (history.size() > maxSize) history.pollLast(); // drop oldest
        System.out.println("Executed: " + command.getDescription());
    }

    public void undo() {
        Command cmd = history.poll();
        if (cmd == null) throw new IllegalStateException("Nothing to undo");
        cmd.undo();
        System.out.println("Undone: " + cmd.getDescription());
    }
}
```

```java
// patterns/state/AccountState.java
public interface AccountState {
    void deposit(Account account, double amount);
    void withdraw(Account account, double amount);
    void freeze(Account account);
    void activate(Account account);
    void close(Account account);
    String getStateName();
}

// patterns/state/ActiveState.java
public class ActiveState implements AccountState {
    @Override
    public void deposit(Account account, double amount) {
        account.setBalance(account.getBalance() + amount);
        System.out.printf("Deposited %.0f. Balance: %.0f%n", amount, account.getBalance());
    }

    @Override
    public void withdraw(Account account, double amount) {
        if (amount > account.getBalance()) throw new IllegalArgumentException("Insufficient funds");
        account.setBalance(account.getBalance() - amount);
        System.out.printf("Withdrew %.0f. Balance: %.0f%n", amount, account.getBalance());
    }

    @Override public void freeze(Account account) { account.setState(new FrozenState()); }
    @Override public void activate(Account account) { System.out.println("Already active"); }
    @Override public void close(Account account) { account.setState(new ClosedState()); }
    @Override public String getStateName() { return "ACTIVE"; }
}

// patterns/state/FrozenState.java
public class FrozenState implements AccountState {
    @Override public void deposit(Account a, double amount) { throw new IllegalStateException("Account frozen"); }
    @Override public void withdraw(Account a, double amount) { throw new IllegalStateException("Account frozen"); }
    @Override public void freeze(Account a) { System.out.println("Already frozen"); }
    @Override public void activate(Account a) { a.setState(new ActiveState()); }
    @Override public void close(Account a) { a.setState(new ClosedState()); }
    @Override public String getStateName() { return "FROZEN"; }
}

// patterns/state/Account.java
public class Account {
    private final String id;
    private double balance;
    private AccountState state;

    public Account(String id, double initialBalance) {
        this.id = id;
        this.balance = initialBalance;
        this.state = new ActiveState();
    }

    // Delegate everything to current state
    public void deposit(double amount) { state.deposit(this, amount); }
    public void withdraw(double amount) { state.withdraw(this, amount); }
    public void freeze() { state.freeze(this); }
    public void activate() { state.activate(this); }
    public void close() { state.close(this); }

    void setState(AccountState state) { this.state = state; }
    void setBalance(double balance) { this.balance = balance; }
    public String getId() { return id; }
    public double getBalance() { return balance; }
    public String getStatus() { return state.getStateName(); }
}
```

```java
// patterns/observer/AccountObserver.java
public interface AccountObserver {
    void onEvent(AccountEvent event);
}

public record AccountEvent(String accountId, String eventType, double amount, String detail) {
    public static AccountEvent deposit(String accountId, double amount) {
        return new AccountEvent(accountId, "DEPOSIT", amount, null);
    }
    public static AccountEvent withdraw(String accountId, double amount) {
        return new AccountEvent(accountId, "WITHDRAW", amount, null);
    }
}

// patterns/observer/NotificationService.java
public class NotificationService {
    private final List<AccountObserver> observers = new ArrayList<>();

    public void addObserver(AccountObserver observer) { observers.add(observer); }
    public void removeObserver(AccountObserver observer) { observers.remove(observer); }

    public void notifyAll(AccountEvent event) {
        observers.forEach(o -> o.onEvent(event));
    }
}
```

```java
// patterns/builder/AccountBuilder.java
public class AccountBuilder {
    private String id;
    private String ownerId;
    private double initialBalance = 0;
    private String currency = "VND";

    public AccountBuilder id(String id) { this.id = id; return this; }
    public AccountBuilder owner(String ownerId) { this.ownerId = ownerId; return this; }
    public AccountBuilder balance(double amount) { this.initialBalance = amount; return this; }
    public AccountBuilder currency(String currency) { this.currency = currency; return this; }

    public Account build() {
        if (id == null || id.isBlank()) throw new IllegalStateException("Account ID required");
        if (ownerId == null || ownerId.isBlank()) throw new IllegalStateException("Owner ID required");
        if (initialBalance < 0) throw new IllegalStateException("Balance cannot be negative");
        return new Account(id, initialBalance);
    }
}
```

---

## 4. Lab Steps

1. **Strategy:** `PaymentProcessor` với `CreditCardPayment` → thử pay vượt limit → switch sang `BankTransferPayment` → pay thành công
2. **Command + Undo:** Execute 3 transfers → undo 2 → verify balances khớp
3. **State:** `Account` → deposit → freeze → withdraw (expect exception) → activate → withdraw (success)
4. **Observer:** Add `EmailObserver` và `SmsObserver` → deposit → cả hai nhận event
5. **Builder:** Thử build Account mà không set id → observe exception message
6. **Pattern combo:** Account với Observer notify khi State thay đổi (ACTIVE → FROZEN)

---

## 5. Checklist tự kiểm tra

- [ ] Strategy: thêm `PayPalPayment` mà không sửa `PaymentProcessor`
- [ ] Command undo chỉ hoạt động sau execute (guard bằng `executed` flag)
- [ ] State: `FrozenState.withdraw()` ném exception, không return false
- [ ] Builder validation trong `build()`, không trong setters
- [ ] Observer: `removeObserver()` hoạt động — observer không nhận event sau khi remove
- [ ] Hiểu tại sao Decorator khác Proxy về intent mặc dù cùng cấu trúc
