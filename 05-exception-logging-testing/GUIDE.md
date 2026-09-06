# Phase 05 — Exception, Logging & Testing · Lý thuyết & Lab

---

## 1. Lý thuyết

### 1.1 Exception Hierarchy

**Checked vs Unchecked — Triết lý:**

**Checked Exception** (`extends Exception`): Compiler force caller phải xử lý (try-catch hoặc throws). Dùng cho **recoverable** errors mà caller có thể và nên handle:
- `IOException`: file không tìm thấy → thông báo user chọn file khác
- `SQLException`: DB lỗi → retry hoặc fallback

**Unchecked Exception** (`extends RuntimeException`): Không force xử lý. Dùng cho **programming errors** (bugs):
- `NullPointerException`: lỗi lập trình, không nên catch để "fix"
- `IllegalArgumentException`: caller truyền argument sai → fix code, không handle runtime

**Tranh luận trong industry:**
Spring, Hibernate, JPA đều dùng RuntimeException vì:
1. Checked exceptions lan truyền qua nhiều layers → `throws IOException` xuất hiện ở controller dù nó không biết gì về IO
2. Checked exceptions không dùng được với lambda (phải try-catch trong lambda — ugly)
3. Thực tế: hầu hết "recoverable" errors đều được handle ở một tầng cao hơn (exception handler) chứ không phải ngay tại caller

**Custom Exception hierarchy (production pattern):**
```
AppException (abstract, RuntimeException)
├── ResourceNotFoundException (404)
├── ValidationException (400)
├── ConflictException (409)
├── ForbiddenException (403)
└── ServiceUnavailableException (503)
```

Lý do: `@RestControllerAdvice` catch `AppException` và map sang HTTP status code. Exception message không expose internal details.

---

### 1.2 Logging Best Practices

**SLF4J là facade, không phải implementation:**
Code của bạn depend on SLF4J API (`Logger`, `LoggerFactory`). Implementation (Logback, Log4j2) được inject at runtime. Có thể swap implementation mà không sửa code.

**Log levels — khi nào dùng gì:**
- `TRACE`: cực kỳ detailed, chỉ bật khi debug cụ thể. `trace("Entry: id={}", id)`
- `DEBUG`: thông tin debug cho dev. Không bật ở prod (performance impact).
- `INFO`: system events bình thường. `info("Order created: orderId={}, userId={}", ...)`
- `WARN`: potential problem nhưng system vẫn chạy. `warn("Retry attempt {} for orderId={}", attempt, id)`
- `ERROR`: error xảy ra, cần attention. `error("Payment failed: orderId={}", id, exception)`

**Structured logging với parameterized messages:**
```java
// ❌ String concatenation — allocates string even if level disabled
log.debug("User: " + userId + " created order: " + orderId);

// ✅ Parameterized — lazy evaluation, no allocation if level disabled
log.debug("User: {} created order: {}", userId, orderId);
```

**MDC (Mapped Diagnostic Context) — correlation:**
MDC là ThreadLocal map — tự động thêm context vào mọi log trong thread đó.
```java
MDC.put("requestId", UUID.randomUUID().toString());
MDC.put("userId", currentUser.getId());
// Tất cả log.info/warn/error sau đây đều có requestId và userId
doBusinessLogic();
MDC.clear(); // PHẢI clear trong finally để tránh leak sang request khác
```

---

### 1.3 Testing Pyramid

```
        /\
       /E2E\           Ít nhất (chậm, fragile, expensive)
      /------\
     /  Integ  \       Vừa (dùng real DB/infra)
    /------------\
   /  Unit Tests  \    Nhiều nhất (nhanh, isolated, cheap)
  /________________\
```

**Unit Test:** Test một class trong isolation, dependencies được mock.
**Integration Test:** Test nhiều layers cùng nhau với real infrastructure (DB, Redis).
**E2E Test:** Test toàn bộ system từ ngoài vào.

**Test Doubles (Mockito):**
- `@Mock`: tạo fake object trả default values (null/0/false)
- `when(...).thenReturn(...)`: configure behavior
- `verify(mock).method(...)`: verify method được gọi
- `@Spy`: wrap real object, chỉ stub specific methods
- `ArgumentCaptor`: capture argument được truyền vào mock để verify

**Test annotations quan trọng (JUnit 5):**
```java
@Test                          // basic test
@ParameterizedTest             // run test nhiều lần với different inputs
@ValueSource(strings = {...})  // values for parameterized
@NullAndEmptySource            // null and "" for parameterized
@CsvSource({"a,1", "b,2"})    // multiple columns
@MethodSource("provider")      // external method provides test data
@EnumSource(value=Status.class) // all enum values
@Nested                        // group related tests
@RepeatedTest(5)               // run 5 times (concurrency test)
@BeforeEach / @AfterEach       // setup/teardown per test
```

**Test naming:** `should_<expectedBehavior>_when_<condition>()` hoặc `<method>_<condition>_<expectedResult>()`

---

### 1.4 Testcontainers — Real Database in Tests

**Tại sao không dùng H2 cho integration test:**
H2 (in-memory DB) khác PostgreSQL về:
- SQL dialect (GENERATED ALWAYS AS IDENTITY vs SERIAL)
- Behavior edge cases (NULL handling, locking)
- Missing features (JSONB, array types, specific functions)

Test với H2 có thể pass nhưng fail ở production với PostgreSQL. Testcontainers start real PostgreSQL trong Docker container → test against real thing.

**Lifecycle:** Container start một lần per test class (`static @Container`), shared across tests → nhanh hơn start-per-test.

---

## 2. Vấn đề thường gặp & Cách fix

### Issue 1: Exception swallowing

```java
// ❌ Exception bị "nuốt" — không biết gì xảy ra
try {
    riskyOperation();
} catch (Exception e) {
    // empty catch — worst pattern
}

// ❌ Log mà không rethrow — nhưng caller không biết có lỗi
try {
    riskyOperation();
} catch (Exception e) {
    log.error("Error occurred"); // missing stack trace! luôn log exception object
}

// ✅ Log với exception object (sẽ in stack trace) + rethrow nếu cần
try {
    riskyOperation();
} catch (DatabaseException e) {
    log.error("Database error during operation: {}", context, e); // 'e' cuối cùng
    throw new ServiceException("Operation failed", e); // wrap với context
}
```

### Issue 2: Mockito verify không chạy

```java
// ❌ verify() PASS dù method không được gọi — tại sao?
// Vì test ném exception trước khi đến verify()
@Test
void test() {
    when(repo.findById(1L)).thenReturn(Optional.empty());
    // service.getUser(1L) sẽ throw ResourceNotFoundException
    service.getUser(1L); // ném exception ở đây
    verify(repo).findById(1L); // không bao giờ chạy!
}

// ✅ Expect exception với assertThrows
@Test
void should_throwNotFound_when_userNotExists() {
    when(repo.findById(1L)).thenReturn(Optional.empty());
    assertThrows(ResourceNotFoundException.class, () -> service.getUser(1L));
    verify(repo).findById(1L); // verify được gọi sau assertThrows
}
```

### Issue 3: Flaky test với Thread.sleep

```java
// ❌ Flaky — Thread.sleep không reliable, slow, non-deterministic
@Test
void should_processAsync() throws Exception {
    service.processAsync(data);
    Thread.sleep(1000); // might not be enough
    verify(mock).save(any());
}

// ✅ Dùng CountDownLatch hoặc CompletableFuture.get() để wait deterministically
@Test
void should_processAsync() throws Exception {
    CountDownLatch latch = new CountDownLatch(1);
    doAnswer(inv -> { latch.countDown(); return null; }).when(mock).save(any());
    service.processAsync(data);
    assertTrue(latch.await(5, SECONDS));
    verify(mock).save(any());
}
```

---

## 3. Code mẫu

```java
// exception/AppException.java
public abstract class AppException extends RuntimeException {
    private final String errorCode;

    protected AppException(String message, String errorCode) {
        super(message);
        this.errorCode = errorCode;
    }

    protected AppException(String message, String errorCode, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
    }

    public String getErrorCode() { return errorCode; }
}

// exception/ResourceNotFoundException.java
public class ResourceNotFoundException extends AppException {
    public ResourceNotFoundException(String resourceType, Object id) {
        super(resourceType + " not found with id: " + id, "RESOURCE_NOT_FOUND");
    }
}

// exception/ValidationException.java
public class ValidationException extends AppException {
    private final Map<String, String> fieldErrors;

    public ValidationException(String message) {
        super(message, "VALIDATION_FAILED");
        this.fieldErrors = Map.of();
    }

    public ValidationException(Map<String, String> fieldErrors) {
        super("Validation failed", "VALIDATION_FAILED");
        this.fieldErrors = fieldErrors;
    }

    public Map<String, String> getFieldErrors() { return fieldErrors; }
}
```

```java
// service/UserService.java
public class UserService {
    private static final Logger log = LoggerFactory.getLogger(UserService.class);
    private final UserRepository repository;

    public UserService(UserRepository repository) {
        this.repository = repository;
    }

    public User createUser(String name, String email) {
        log.info("Creating user: email={}", email); // không log password!

        validateEmail(email);

        if (repository.findByEmail(email).isPresent()) {
            log.warn("Duplicate email registration attempt: email={}", email);
            throw new ConflictException("Email already registered: " + email);
        }

        User user = new User(name, email);
        User saved = repository.save(user);
        log.info("User created: userId={}, email={}", saved.getId(), email);
        return saved;
    }

    public User getUser(Long id) {
        return repository.findById(id)
            .orElseThrow(() -> {
                log.warn("User not found: id={}", id);
                return new ResourceNotFoundException("User", id);
            });
    }

    private void validateEmail(String email) {
        if (email == null || !email.matches("^[A-Za-z0-9+_.-]+@.+$")) {
            throw new ValidationException("Invalid email format: " + email);
        }
    }
}
```

```java
// test/UserServiceTest.java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock UserRepository repository;
    @InjectMocks UserService service;

    @Nested
    class CreateUser {

        @Test
        void should_createUser_when_validData() {
            // Arrange
            String name = "Alice", email = "alice@example.com";
            User expected = new User(1L, name, email);
            when(repository.findByEmail(email)).thenReturn(Optional.empty());
            when(repository.save(any(User.class))).thenReturn(expected);

            // Act
            User result = service.createUser(name, email);

            // Assert
            assertThat(result.getId()).isEqualTo(1L);
            assertThat(result.getEmail()).isEqualTo(email);

            // Verify interactions
            verify(repository).findByEmail(email);
            ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
            verify(repository).save(captor.capture());
            assertThat(captor.getValue().getName()).isEqualTo(name);
        }

        @ParameterizedTest(name = "invalid email: [{0}]")
        @NullAndEmptySource
        @ValueSource(strings = {"not-an-email", "missing@", "@no-local", "spaces in@email.com"})
        void should_throwValidation_when_invalidEmail(String email) {
            assertThrows(ValidationException.class, () -> service.createUser("Test", email));
            verifyNoInteractions(repository); // DB không được gọi
        }

        @Test
        void should_throwConflict_when_emailAlreadyExists() {
            when(repository.findByEmail("exists@test.com"))
                .thenReturn(Optional.of(new User("Existing", "exists@test.com")));

            ConflictException ex = assertThrows(ConflictException.class,
                () -> service.createUser("New", "exists@test.com"));
            assertThat(ex.getMessage()).contains("already registered");
        }
    }

    @Nested
    class GetUser {

        @Test
        void should_returnUser_when_exists() {
            User user = new User(42L, "Bob", "bob@test.com");
            when(repository.findById(42L)).thenReturn(Optional.of(user));

            assertThat(service.getUser(42L).getName()).isEqualTo("Bob");
        }

        @Test
        void should_throwNotFound_when_notExists() {
            when(repository.findById(99L)).thenReturn(Optional.empty());

            ResourceNotFoundException ex = assertThrows(ResourceNotFoundException.class,
                () -> service.getUser(99L));
            assertThat(ex.getMessage()).contains("99");
        }
    }
}
```

---

## 4. Lab Steps

1. **Exception hierarchy:** Tạo `AppException → ConflictException`, ném từ service, catch trong main và print `errorCode`
2. **MDC:** Add `MDC.put("requestId", ...)` trong test setup, verify log output có requestId
3. **@Nested:** Group "happy path" tests và "error cases" vào `@Nested` classes riêng
4. **ArgumentCaptor:** Capture User object được save, assert từng field
5. **@ParameterizedTest:** Test email validation với 6 invalid formats
6. **Testcontainers:** Setup `@DataJpaTest @Testcontainers` với PostgreSQLContainer

---

## 5. Checklist tự kiểm tra

- [ ] `log.error("msg", e)` — luôn pass exception object để có stack trace
- [ ] `MDC.clear()` trong `finally` block — không thể bỏ qua
- [ ] `verify()` sau `assertThrows` — không trước (test sẽ fail hoặc không chạy đến verify)
- [ ] Custom exceptions có `errorCode` để map sang HTTP status code
- [ ] `@InjectMocks` creates real service với mocked dependencies injected
- [ ] `verifyNoInteractions(repo)` khi validation fail trước khi repo được gọi
- [ ] Testcontainers: `@DynamicPropertySource` đặt DB URL từ container runtime port
