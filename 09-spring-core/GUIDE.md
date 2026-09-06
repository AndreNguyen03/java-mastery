# Phase 09 — Spring Core · Lý thuyết & Lab

---

## 1. Lý thuyết

### 1.0 BeanFactory vs ApplicationContext

**BeanFactory** — interface cơ bản nhất của IoC container:
- Lazy initialization: bean chỉ được tạo khi `getBean()` được gọi
- Không có annotation processing, event publishing, AOP auto-proxy
- Chỉ dùng trong environments cực kỳ resource-constrained (embedded, mobile)

**ApplicationContext** — extends BeanFactory với nhiều enterprise features:
- **Eager initialization** của singleton beans khi startup (phát hiện lỗi sớm hơn)
- **MessageSource** — i18n support
- **ApplicationEventPublisher** — publish/listen events
- **ResourceLoader** — load files từ classpath/filesystem/URL
- **BeanPostProcessor** tự động registered — AOP, `@Autowired`, `@Value`
- **Environment** — profiles và property resolution

```java
// BeanFactory: manual, verbose
BeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions("classpath:beans.xml");

// ApplicationContext: convenient, feature-rich (bạn luôn dùng cái này)
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
UserService service = ctx.getBean(UserService.class);

// Trong Spring Boot: ApplicationContext được tạo bởi SpringApplication.run()
@SpringBootApplication
public class Main {
    public static void main(String[] args) {
        ApplicationContext ctx = SpringApplication.run(Main.class, args);
        // ctx available nhưng không cần dùng trực tiếp — DI tự inject
    }
}
```

**Tóm tắt quyết định:** Luôn dùng `ApplicationContext`. `BeanFactory` chỉ là implementation detail.

---

### 1.00 Spring Profiles — Cấu hình theo môi trường

**Vấn đề:** Dev, staging, production cần config khác nhau (DB URL, credentials, feature flags). Hard-coding gây security risk và deployment nightmare.

```java
// @Profile — bean chỉ được tạo nếu profile active
@Configuration
@Profile("dev")
public class DevDataSourceConfig {
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build(); // in-memory H2 for dev
    }
}

@Configuration
@Profile("prod")
public class ProdDataSourceConfig {
    @Bean
    public DataSource dataSource(Environment env) {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl(env.getRequiredProperty("db.url"));
        ds.setUsername(env.getRequiredProperty("db.username"));
        ds.setPassword(env.getRequiredProperty("db.password"));
        return ds; // real PostgreSQL for prod
    }
}

// @Profile với NOT operator (Spring 3.1+)
@Bean
@Profile("!prod") // active trên mọi profile NGOẠI TRỪ prod
public DataSeeder testDataSeeder() {
    return new TestDataSeeder(); // seed test data on dev/staging only
}
```

**Activate profile:**
```yaml
# application.yml (default)
spring:
  profiles:
    active: dev  # set default profile

# application-dev.yml (loaded when profile=dev)
spring.datasource.url: jdbc:h2:mem:testdb

# application-prod.yml (loaded when profile=prod)
spring.datasource.url: jdbc:postgresql://prod-db:5432/appdb
```

```bash
# Override via env variable (12-factor app pattern)
SPRING_PROFILES_ACTIVE=prod java -jar app.jar
# Or JVM arg:
java -Dspring.profiles.active=prod -jar app.jar
```

```java
// In tests:
@SpringBootTest
@ActiveProfiles("test") // activates test profile
class UserServiceTest {
    // beans with @Profile("test") active
}
```

**Profile hierarchy:** `application.yml` → `application-{profile}.yml`. Profile-specific properties **override** base application.yml.

---

### 1.000 @ConfigurationProperties — Typed Configuration

**Vấn đề với `@Value`:**
```java
@Value("${mail.host}") String host;
@Value("${mail.port}") int port;
@Value("${mail.username}") String username;
// Scattered — hard to see all mail configs together
// No type validation until runtime
// No IDE autocomplete without extra setup
```

**`@ConfigurationProperties` — typed, grouped config:**
```java
// application.yml
app:
  mail:
    host: smtp.gmail.com
    port: 587
    username: noreply@example.com
    connection-timeout: PT30S  # Java Duration format
    retry:
      max-attempts: 3
      backoff-ms: 1000
```

```java
// Config class — mapped from YAML
@ConfigurationProperties(prefix = "app.mail")
@Validated // enable Bean Validation on properties
public record MailProperties(
    @NotBlank String host,
    @Min(1) @Max(65535) int port,
    @NotBlank String username,
    @DurationUnit(ChronoUnit.SECONDS) Duration connectionTimeout,
    RetryConfig retry
) {
    public record RetryConfig(
        @Positive int maxAttempts,
        @Positive long backoffMs
    ) {}
}

// Enable in @Configuration or @SpringBootApplication:
@SpringBootApplication
@ConfigurationPropertiesScan // scans for @ConfigurationProperties beans
public class Application {}

// Or explicitly:
@Configuration
@EnableConfigurationProperties(MailProperties.class)
public class MailConfig {}

// Inject like any other bean:
@Service
public class MailService {
    private final MailProperties props;
    
    public MailService(MailProperties props) {
        this.props = props;
    }
    
    public void send(String to, String subject, String body) {
        // props.host(), props.port(), etc.
    }
}
```

**Metadata — IDE autocomplete:**
Add `spring-boot-configuration-processor` dependency → annotation processor generates `META-INF/spring-configuration-metadata.json` → IntelliJ shows autocomplete and docs for your properties.

---

### 1.1 IoC & Dependency Injection

**Vấn đề không có IoC:**
```java
// ❌ Tight coupling — UserService tự tạo dependencies
class UserService {
    private final UserRepository repo = new JdbcUserRepository(); // hard-coded impl
    private final EmailService email = new SmtpEmailService("smtp.gmail.com"); // config embedded

    // Muốn test → phải hit real DB và real email server!
    // Muốn swap sang mock → phải sửa code
}
```

**IoC (Inversion of Control):** Thay vì class tự tạo dependencies, dependencies được **inject vào từ ngoài**. Control flow được đảo ngược — "don't call us, we'll call you".

**DI Styles:**
```java
// Constructor injection (preferred)
class UserService {
    private final UserRepository repo; // can't be changed after construction (immutable)
    
    public UserService(UserRepository repo) { // dependency is explicit
        this.repo = repo;
    }
}
// Advantages: immutable, easy to test (just pass mock), required deps obvious, no null risk

// Setter injection (optional dependencies)
class UserService {
    private EmailService emailService; // might be null
    
    public void setEmailService(EmailService e) { this.emailService = e; }
}

// Field injection (avoid — Spring's @Autowired on field)
@Autowired private UserRepository repo; // hidden dependency, can't inject in unit test without reflection
```

---

### 1.2 Spring ApplicationContext — IoC Container

**ApplicationContext là gì:**
Là container quản lý lifecycle của beans — tạo, wire dependencies, configure, destroy. Khi bạn call `getBean("userService")`, container:
1. Check bean đã được khởi tạo chưa (nếu singleton)
2. Nếu chưa: instantiate, inject dependencies, call `@PostConstruct`
3. Trả lại bean

**Bean Definition vs Bean Instance:**
- Definition: metadata (class, scope, constructor args, etc.) — giống "recipe"
- Instance: object được tạo từ definition

**Bean Scopes:**
- `singleton` (default): một instance per container — shared. Stateless services nên là singleton.
- `prototype`: instance mới mỗi lần `getBean()`. Có state nên là prototype.
- `request`: (web) instance per HTTP request
- `session`: (web) instance per HTTP session

**Khi nào singleton nguy hiểm:**
```java
@Service // singleton
class OrderService {
    private List<Order> pendingOrders = new ArrayList<>(); // MUTABLE STATE IN SINGLETON!
    
    public void addOrder(Order o) { pendingOrders.add(o); } // race condition!
}
```
Singleton + mutable state = thread-safety issues. Singleton nên stateless, hoặc state phải thread-safe.

---

### 1.3 Bean Lifecycle

```
Spring Container Startup:
1. Read configuration (@Configuration, XML, component scan)
2. Instantiate beans (constructor)
3. Populate properties (setters, @Autowired)
4. BeanPostProcessor.postProcessBeforeInitialization()
5. @PostConstruct / afterPropertiesSet() / init-method
6. BeanPostProcessor.postProcessAfterInitialization()
7. Bean ready to use

Container Shutdown:
8. @PreDestroy / destroy() / destroy-method
9. GC cleanup
```

**`@PostConstruct` use cases:**
- Validate configuration: "check DB connection works"
- Initialize state from DB/config
- Register listeners

**`BeanPostProcessor`:**
Intercept EVERY bean creation. Spring dùng BPP internally cho: `@Autowired`, `@Transactional`, `@Async`, AOP proxies. Custom BPP: logging, validation, tracing.

---

### 1.4 AOP — Aspect-Oriented Programming

**Cross-cutting concerns:** Chức năng cắt ngang nhiều modules — logging, security, transaction, caching. Nếu tung vào mỗi class:
- Code bị lặp lại khắp nơi
- Business logic bị pha trộn với infrastructure concerns
- Khó thay đổi (phải sửa hàng trăm places)

**AOP concepts:**
- **Aspect:** module chứa cross-cutting logic
- **Pointcut:** biểu thức định nghĩa "chỗ nào" advice được apply (which methods)
- **Advice:** "điều gì" xảy ra tại pointcut (before/after/around)
- **JoinPoint:** một điểm cụ thể trong execution (method call)
- **Weaving:** quá trình áp dụng aspect vào target code

**Proxy models:**
Spring AOP dùng **proxy pattern** — tạo wrapper object implement cùng interface:
- JDK Dynamic Proxy: target phải implement interface
- CGLIB Proxy: subclass target class (dùng khi không có interface)

**Self-invocation problem — hiểu để debug:**
```java
@Service
class OrderService {
    public void createOrder(Order o) {
        validateOrder(o); // ← direct call, NOT through proxy!
        save(o);
    }
    
    @Transactional // ← annotation on this method
    void validateOrder(Order o) {
        // Transaction KHÔNG được start!
        // Vì createOrder gọi validateOrder trực tiếp (this.validateOrder)
        // Không phải proxy.validateOrder
    }
}
```

**Fix:** Inject `self` (autowire the proxy) hoặc refactor ra class khác.

---

### 1.5 Spring Events

**Loose coupling với events:**
Thay vì `OrderService` gọi trực tiếp `EmailService`, `InventoryService`, `AuditService` → `OrderService` chỉ publish event. Ai quan tâm thì listen. Dễ thêm listener mới mà không sửa OrderService.

```java
// Publisher
@Service class OrderService {
    @Autowired ApplicationEventPublisher eventPublisher;
    
    public Order createOrder(Order order) {
        Order saved = orderRepo.save(order);
        eventPublisher.publishEvent(new OrderCreatedEvent(saved)); // fire and forget
        return saved;
    }
}

// Listener 1
@Component class EmailListener {
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        sendConfirmationEmail(event.getOrder());
    }
}

// Listener 2 — async processing
@Component class InventoryListener {
    @EventListener
    @Async // run in separate thread
    public void onOrderCreated(OrderCreatedEvent event) {
        reserveInventory(event.getOrder());
    }
}
```

**Synchronous by default:** `@EventListener` chạy cùng thread với publisher. Nếu listener ném exception → publisher method fails. Dùng `@Async` để async.

---

## 2. Vấn đề thường gặp & Cách fix

### Issue 1: Circular dependency

```
UserService → PasswordEncoder → UserService (circular!)
```

**Fix:**
```java
// Option 1: Setter injection (circular deps ok với setter)
@Service class A {
    private B b;
    @Autowired void setB(B b) { this.b = b; } // Spring resolves after both created
}

// Option 2: @Lazy — one dep created lazily
@Autowired @Lazy private B b; // B proxy created first, actual B created on first use

// Option 3 (best): redesign — usually circular dep = design issue
// Extract shared interface or third class
```

### Issue 2: `@Transactional` không hoạt động

**5 nguyên nhân phổ biến:**

1. **Self-invocation** (đã đề cập)
2. **Method không phải public** — Spring AOP chỉ proxy public methods
3. **Exception bị catch trước khi rethrow** — transaction không rollback
4. **RuntimeException vs checked** — mặc định chỉ rollback RuntimeException

```java
// ❌ Transaction không rollback vì IOException bị checked
@Transactional
public void doWork() throws IOException {
    repo.save(entity);
    throw new IOException("disk full"); // NO rollback by default!
}

// ✅ Specify rollback for checked exception
@Transactional(rollbackFor = {IOException.class})
public void doWork() throws IOException { ... }
```

5. **Bean không trong Spring context** (new MyService() không phải proxy)

### Issue 3: Prototype bean trong Singleton

```java
// ❌ Prototype bean bị injected once into singleton → always same instance!
@Component @Scope("prototype")
class ShoppingCart { ... }

@Service // singleton
class CheckoutService {
    @Autowired ShoppingCart cart; // INJECTED ONCE — always same cart for all users!
}

// ✅ Option 1: inject ApplicationContext, call getBean each time
@Service class CheckoutService {
    @Autowired ApplicationContext ctx;
    public void checkout() {
        ShoppingCart cart = ctx.getBean(ShoppingCart.class); // new instance each time
    }
}

// ✅ Option 2: Provider
@Autowired Provider<ShoppingCart> cartProvider;
ShoppingCart cart = cartProvider.get(); // new instance
```

---

## 3. Code mẫu — Spring Core Lab

```java
// config/AppConfig.java
@Configuration
@ComponentScan("com.nguyenngoc.phase09")
public class AppConfig {

    // Explicit bean definition — when can't annotate class (third-party)
    @Bean
    public Clock clock() {
        return Clock.systemDefaultZone(); // injectable clock — makes testing easier
    }
}
```

```java
// lifecycle/LifecycleBean.java
@Component
public class LifecycleBean implements InitializingBean, DisposableBean {
    private static final Logger log = LoggerFactory.getLogger(LifecycleBean.class);

    @PostConstruct  // runs after injection, before bean ready
    public void init() {
        log.info("@PostConstruct: bean ready, validating config...");
    }

    @Override
    public void afterPropertiesSet() {
        log.info("InitializingBean.afterPropertiesSet(): all properties injected");
    }

    @PreDestroy // runs before container shutdown
    public void cleanup() {
        log.info("@PreDestroy: releasing resources...");
    }

    @Override
    public void destroy() {
        log.info("DisposableBean.destroy()");
    }
}
```

```java
// aop/LoggingAspect.java
@Aspect
@Component
public class LoggingAspect {

    private static final Logger log = LoggerFactory.getLogger(LoggingAspect.class);

    /**
     * Pointcut: matches any method in service package
     * Expressions: execution, within, @annotation, bean
     */
    @Pointcut("within(com.nguyenngoc.phase09.service..*)")
    public void serviceLayer() {}

    /**
     * @Around: most powerful — controls whether to proceed
     * Can modify args, return value, suppress exception
     */
    @Around("serviceLayer()")
    public Object logExecution(ProceedingJoinPoint pjp) throws Throwable {
        String methodName = pjp.getSignature().toShortString();
        log.info("→ {}", methodName);
        long start = System.currentTimeMillis();

        try {
            Object result = pjp.proceed(); // execute actual method
            log.info("← {} completed in {}ms", methodName, System.currentTimeMillis() - start);
            return result;
        } catch (Throwable ex) {
            log.error("← {} failed after {}ms: {}", methodName, System.currentTimeMillis() - start, ex.getMessage());
            throw ex; // rethrow — don't swallow
        }
    }

    /**
     * @Annotation pointcut: apply only to methods annotated with @Audited
     */
    @Before("@annotation(com.nguyenngoc.phase09.aop.Audited)")
    public void auditMethodCall(JoinPoint jp) {
        log.info("AUDIT: {}", jp.getSignature().toShortString());
    }
}
```

```java
// aop/Audited.java
@Retention(RetentionPolicy.RUNTIME) // needed for reflection at runtime
@Target(ElementType.METHOD)         // only applies to methods
public @interface Audited {
    String action() default ""; // optional metadata
}
```

```java
// events/OrderEvents.java
public record OrderCreatedEvent(String orderId, String userId, double amount) {}

@Service
public class OrderService {
    private final ApplicationEventPublisher eventPublisher;
    
    public OrderService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    @Audited(action = "CREATE_ORDER")
    public String createOrder(String userId, double amount) {
        String orderId = UUID.randomUUID().toString().substring(0, 8);
        // ... save order ...
        eventPublisher.publishEvent(new OrderCreatedEvent(orderId, userId, amount));
        return orderId;
    }
}

@Component
public class NotificationListener {
    private static final Logger log = LoggerFactory.getLogger(NotificationListener.class);

    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        log.info("📧 Sending confirmation for orderId={}", event.orderId());
    }
}

@Component
public class AuditListener {
    private static final Logger log = LoggerFactory.getLogger(AuditListener.class);

    @EventListener
    @Order(1) // run before other listeners
    public void onOrderCreated(OrderCreatedEvent event) {
        log.info("📝 Audit: order {} created by user {}", event.orderId(), event.userId());
    }
}
```

```java
// test/DITest.java
@SpringJUnitConfig(AppConfig.class)
class DITest {

    @Autowired OrderService orderService;
    @Autowired ApplicationContext context;

    @Test
    void should_list_all_beans() {
        String[] names = context.getBeanDefinitionNames();
        System.out.println("Total beans: " + names.length);
        Arrays.stream(names).sorted().forEach(System.out::println);
    }

    @Test
    void should_singleton_return_same_instance() {
        OrderService a = context.getBean(OrderService.class);
        OrderService b = context.getBean(OrderService.class);
        assertSame(a, b); // same object — singleton
    }

    @Test
    void should_create_order_and_fire_event() {
        // OrderService has AOP proxy applied via @Audited
        // Event listeners are called during createOrder
        String orderId = orderService.createOrder("user-1", 50000);
        assertNotNull(orderId);
    }
}
```

---

## 4. Lab Steps

1. **Component scan:** Add `@ComponentScan`, list all beans → thấy Spring infrastructure beans + your beans
2. **Singleton vs Prototype:** `getBean(Service.class) == getBean(Service.class)` → true cho singleton, false cho prototype
3. **Lifecycle order:** Bật logging DEBUG, startup → observe init order: constructor → `@Autowired` → `afterPropertiesSet` → `@PostConstruct`
4. **AOP around:** `LoggingAspect` log method entry/exit → verify output
5. **Self-invocation:** `OrderService.createOrder()` gọi internal `@Transactional` method → verify transaction không start (enable `spring.jpa.show-sql=true`)
6. **Events:** Thêm `InventoryListener`, verify cả 3 listeners nhận event sau `createOrder`

---

## 5. Checklist tự kiểm tra

- [ ] Constructor injection preferred over field injection (`@Autowired` trên field)
- [ ] Singleton beans không có mutable instance fields (thread safety)
- [ ] `@PostConstruct` chạy sau `@Autowired` — có thể dùng injected dependencies
- [ ] AOP self-invocation: `this.method()` không qua proxy → advice không chạy
- [ ] `@Transactional` mặc định rollback `RuntimeException`, không phải `Exception`
- [ ] `@EventListener` synchronous — listener exception ảnh hưởng publisher method
