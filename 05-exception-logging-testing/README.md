# Phase 05 — Exception, Logging & Testing

> **Mục tiêu:** Biết test code và debug vấn đề một cách có hệ thống. Một feature chưa có test = chưa done.

---

## Part 1 — Exception

### Exception Hierarchy
```
Throwable
├── Error                   ← JVM-level, không catch
│   ├── OutOfMemoryError
│   └── StackOverflowError
└── Exception
    ├── RuntimeException    ← Unchecked (không bắt buộc handle)
    │   ├── NullPointerException
    │   ├── IllegalArgumentException
    │   ├── IllegalStateException
    │   └── IndexOutOfBoundsException
    └── IOException         ← Checked (bắt buộc handle hoặc declare)
        ├── FileNotFoundException
        └── ...
```

### Checked vs Unchecked
| | Checked | Unchecked |
|--|---------|-----------|
| Compile-time | Bắt buộc handle | Không bắt buộc |
| Khi nào dùng | Recoverable failure (file not found, network timeout) | Programming error (null, invalid arg) |
| Ví dụ | `IOException`, `SQLException` | `NPE`, `IllegalArgumentException` |

**Tranh cãi:** Java hiện đại (và Spring) ưu tiên Unchecked cho business exceptions vì:
- Checked exceptions phá vỡ encapsulation khi propagate
- Caller thường không biết cách handle → catch-and-swallow

### Custom Exception
```java
// Hierarchy cho business exceptions
public class AppException extends RuntimeException {
    private final ErrorCode code;
    public AppException(ErrorCode code, String message) {
        super(message);
        this.code = code;
    }
}

public class ResourceNotFoundException extends AppException { ... }
public class ValidationException extends AppException { ... }
public class ConflictException extends AppException { ... }
```

### try-with-resources
```java
// Resource tự động đóng dù exception xảy ra
try (var conn = dataSource.getConnection();
     var stmt = conn.prepareStatement(sql)) {
    // ...
} // conn và stmt đều được close() tự động
```
Resource phải implement `AutoCloseable`.

### Anti-patterns
```java
// Sai: swallow exception
try { ... } catch (Exception e) { }

// Sai: catch quá rộng
try { ... } catch (Exception e) { log.error("error"); }

// Sai: exception as control flow
try { Integer.parseInt(s); return true; } catch (NumberFormatException e) { return false; }
// Đúng: dùng regex hoặc StringUtils.isNumeric()
```

---

## Part 2 — Logging

### SLF4J + Logback
```
SLF4J = API (interface) — code của bạn import cái này
Logback = Implementation — được swap bằng Log4j2, java.util.logging…
```

```java
private static final Logger log = LoggerFactory.getLogger(MyService.class);
// hoặc với Lombok:
@Slf4j
public class MyService { }
```

### Log Levels
| Level | Dùng khi |
|-------|---------|
| `TRACE` | Rất chi tiết, từng step — chỉ dùng khi debug intensive |
| `DEBUG` | Chi tiết, hữu ích khi develop |
| `INFO` | Milestone quan trọng: app start, request nhận được, job hoàn thành |
| `WARN` | Có vấn đề nhưng app vẫn chạy được |
| `ERROR` | Lỗi nghiêm trọng, cần attention |

### Structured Logging
```java
// Sai: string concatenation
log.info("Processing order " + orderId + " for user " + userId);

// Đúng: parameterized (không tạo String nếu level bị filter)
log.info("Processing order {} for user {}", orderId, userId);

// Tốt nhất: structured với key-value (searchable trong ELK/Splunk)
log.info("order.processing", kv("orderId", orderId), kv("userId", userId));
```

### Correlation ID (Trace ID)
```java
// Trong filter/interceptor:
MDC.put("correlationId", UUID.randomUUID().toString());
// Log tự động include correlationId trong mọi log statement của request đó
MDC.clear(); // sau khi request xong
```

---

## Part 3 — Testing

### Testing Pyramid
```
        /\
       /E2E\          ← ít, chậm, đắt (Selenium, Playwright)
      /------\
     /Integr. \       ← trung bình (DB, HTTP, Kafka)
    /----------\
   / Unit Tests \     ← nhiều, nhanh, rẻ (JUnit + Mockito)
  /--------------\
```

### JUnit 5 — Cú pháp cần biết
```java
@Test
void should_returnEmpty_when_userNotFound() {    // tên test mô tả behavior
    // Arrange
    var repo = mock(UserRepository.class);
    when(repo.findById(99L)).thenReturn(Optional.empty());
    var service = new UserService(repo);

    // Act
    Optional<User> result = service.findUser(99L);

    // Assert
    assertThat(result).isEmpty();
}

@ParameterizedTest
@ValueSource(strings = {"", " ", "\t"})
void should_throwException_when_nameBlank(String name) {
    assertThatThrownBy(() -> new User(name))
        .isInstanceOf(IllegalArgumentException.class);
}

@Nested class WhenUserIsActive { ... }   // nhóm test theo context
@DisplayName("UserService")              // tên đẹp hơn trong report
```

### Mockito — Test Doubles
| Double | Mô tả | Mockito |
|--------|-------|---------|
| Mock | Fake object, verify interactions | `mock()` |
| Stub | Trả về giá trị định sẵn | `when(...).thenReturn(...)` |
| Spy | Real object + verify | `spy()` |
| Verify | Kiểm tra method đã được gọi | `verify(mock).method(args)` |

```java
// Stub
when(userRepo.findById(1L)).thenReturn(Optional.of(user));

// Verify behavior
verify(emailService, times(1)).sendWelcome(user.getEmail());
verify(emailService, never()).sendWelcome(anyString());

// Capture argument
ArgumentCaptor<Email> captor = ArgumentCaptor.forClass(Email.class);
verify(emailService).send(captor.capture());
assertThat(captor.getValue().subject()).isEqualTo("Welcome");
```

### Integration Test với Testcontainers
```java
@Testcontainers
@SpringBootTest
class OrderRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
    }

    @Test
    void should_persistOrder() { ... }
}
```

### Naming Convention
```
// Pattern: should_[expectedBehavior]_when_[condition]
should_returnUser_when_idExists()
should_throwNotFound_when_idMissing()
should_sendEmail_when_orderConfirmed()
```

---

## Project — Thêm test cho Expense Tracker hoặc Banking Engine

**Không tạo project mới.** Lấy project từ phase 01 hoặc 04 và viết test đầy đủ.

```
src/test/java/
├── unit/
│   ├── ExpenseServiceTest.java      ← unit test với Mockito
│   ├── ExpenseValidatorTest.java    ← unit test logic
│   └── CategoryTest.java
├── integration/
│   └── ExpenseRepositoryTest.java   ← test với real file I/O
└── architecture/
    └── NamingConventionTest.java    ← kiểm tra test coverage hợp lý
```

**Yêu cầu:**
1. Mỗi public method có ít nhất 1 happy path test + 1 sad path test
2. Dùng `@ParameterizedTest` cho các boundary case
3. Dùng `@Nested` để nhóm test theo scenario
4. Mock external dependencies (file, clock)
5. Test custom exceptions được throw đúng

---

## Checklist — 6 câu hỏi

Ví dụ với `Mock vs Stub`:

| # | Câu hỏi | Trả lời |
|---|---------|---------|
| 1 | Là gì? | Test doubles — fake objects thay thế real dependencies |
| 2 | Giải quyết gì? | Isolate unit under test khỏi external deps (DB, email, time) |
| 3 | Hoạt động thế nào? | Mockito tạo runtime proxy; stub trả về giá trị định sẵn; mock verify calls |
| 4 | Khi nào dùng? | Unit test; khi real dep chậm/flaky/có side effect |
| 5 | Khi nào không? | Integration test — test real DB với Testcontainers thay |
| 6 | Debug thế nào? | `UnnecessaryStubbingException` → stub không được dùng; `WantedButNotInvoked` → method không được gọi |

---

## Run

```bash
mvn test -pl phase-05-exception-logging-testing
```
