# Phase 09 — Spring Core (no Boot)

> **Mục tiêu:** Hiểu Spring IoC và AOP từ gốc. Không dùng Spring Boot — quan sát Spring ở level thấp nhất để Spring Boot không còn là magic.

---

## Part 1 — IoC & Dependency Injection

### Inversion of Control
```
Traditional:
  MyService service = new MyService(new MySqlRepository());

IoC:
  @Component class MyService { @Autowired MySqlRepository repo; }
  // Spring tạo và inject — bạn không gọi new
```

**Tại sao IoC tốt hơn:**
- Loose coupling: MyService không biết concrete class của repository
- Testable: dễ inject mock
- Configuration: swap implementation không cần sửa code

### DI Types
```java
// Constructor injection (RECOMMENDED)
@Component
class OrderService {
    private final OrderRepository repo;
    public OrderService(OrderRepository repo) { this.repo = repo; }
}

// Field injection (KHÔNG recommended — không testable without Spring)
@Autowired private OrderRepository repo;

// Setter injection (dùng cho optional deps)
@Autowired(required = false)
public void setAuditService(AuditService audit) { ... }
```

---

## Part 2 — ApplicationContext & BeanFactory

```
BeanFactory         ← lazy, tối giản
    ↑
ApplicationContext   ← eager, full-featured (dùng cái này)
    ├── ClassPathXmlApplicationContext
    ├── AnnotationConfigApplicationContext  ← Java config
    └── FileSystemXmlApplicationContext
```

```java
// Dùng Java config (không XML)
@Configuration
@ComponentScan("com.nguyenngoc.phase09")
class AppConfig {
    @Bean
    DataSource dataSource() { ... }
}

ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
OrderService service = ctx.getBean(OrderService.class);
```

---

## Part 3 — Bean Definition & Lifecycle

### Bean Scopes
| Scope | Tạo khi | Instances |
|-------|---------|-----------|
| `singleton` | Container startup (default) | 1 per container |
| `prototype` | Every `getBean()` call | N per container |
| `request` | Each HTTP request (web only) | 1 per request |
| `session` | Each HTTP session (web only) | 1 per session |

### Bean Lifecycle
```
Container starts
    ↓
BeanDefinition loaded (class, scope, dependencies)
    ↓
Instantiate (constructor)
    ↓
Populate properties (@Autowired, @Value)
    ↓
BeanPostProcessor.postProcessBeforeInitialization()
    ↓
@PostConstruct / InitializingBean.afterPropertiesSet()
    ↓
BeanPostProcessor.postProcessAfterInitialization()  ← AOP proxy created HERE
    ↓
Bean READY
    ↓
@PreDestroy / DisposableBean.destroy()
    ↓
Container shutdown
```

### BeanPostProcessor
```java
@Component
class LoggingBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        log.info("Bean created: {}", beanName);
        return bean; // QUAN TRỌNG: trả về bean (hoặc wrapper)
    }
}
```
**Đây là nơi Spring tạo AOP proxy.**

---

## Part 4 — AOP

### Tại sao AOP tồn tại
```
// Không có AOP — cross-cutting concern lặp ở mọi nơi
class OrderService {
    void createOrder(...) {
        log.info("before");  // logging
        checkPermission();    // security
        beginTransaction();   // transaction
        // ... actual business logic
        commitTransaction();
        log.info("after");
    }
}

// Với AOP — tách hẳn
@Aspect @Component
class LoggingAspect {
    @Around("@annotation(Loggable)")
    Object log(ProceedingJoinPoint pjp) throws Throwable {
        log.info("Before: {}", pjp.getSignature());
        Object result = pjp.proceed(); // gọi actual method
        log.info("After: {}", pjp.getSignature());
        return result;
    }
}
```

### AOP Concepts
| Term | Nghĩa |
|------|-------|
| **Aspect** | Module chứa cross-cutting logic |
| **Advice** | Hành động tại điểm cắt (Before, After, Around, AfterReturning, AfterThrowing) |
| **Pointcut** | Expression xác định where to apply (method, annotation, class) |
| **Join Point** | Điểm trong execution (method call, field access) |
| **Proxy** | Object wrapping target — JDK Dynamic Proxy (interface) hoặc CGLIB (class) |

### Proxy Models
```
JDK Dynamic Proxy:
  Interface → Proxy implements same interface → delegate to target

CGLIB:
  Subclass → Proxy extends target class → override methods

Spring dùng: JDK proxy nếu bean implement interface, CGLIB nếu không
```

**Gotcha:** `@Transactional` / `@Async` / AOP không hoạt động khi gọi internal (self-invocation) — vì gọi trực tiếp không qua proxy.

---

## Part 5 — Spring Events

```java
// Event
record OrderCreatedEvent(Order order) implements ApplicationEvent {}

// Publisher
@Component
class OrderService {
    @Autowired ApplicationEventPublisher publisher;
    void createOrder(Order order) {
        // ... save
        publisher.publishEvent(new OrderCreatedEvent(order));
    }
}

// Listener
@Component
class EmailListener {
    @EventListener
    void onOrderCreated(OrderCreatedEvent event) {
        sendEmail(event.order());
    }
}
```

---

## Part 6 — Environment & Profiles

```java
@Profile("prod")
@Component
class ProdDataSource implements DataSource { ... }

@Profile("test")
@Component
class InMemoryDataSource implements DataSource { ... }
```

```bash
java -Dspring.profiles.active=prod -jar app.jar
```

---

## Project — Spring Core Lab (Experiments)

**Không build application lớn. Viết nhiều experiment nhỏ để quan sát Spring.**

```
src/
├── experiment01_di/
│   ├── ConstructorInjection.java
│   ├── FieldInjectionProblem.java    ← tại sao field injection xấu
│   └── CircularDependencyDemo.java   ← Spring detect và báo lỗi
├── experiment02_scope/
│   ├── SingletonVsPrototype.java     ← verify same instance vs different
│   └── PrototypeInSingleton.java     ← problem + solution (ObjectProvider)
├── experiment03_lifecycle/
│   ├── BeanLifecycleObserver.java    ← log mọi lifecycle event
│   └── CustomBPP.java                ← custom BeanPostProcessor
├── experiment04_aop/
│   ├── LoggingAspect.java            ← @Around logging
│   ├── TimingAspect.java             ← measure execution time
│   ├── SelfInvocationProblem.java    ← AOP không work với self-call
│   └── ProxyInspector.java           ← print proxy class name
├── experiment05_events/
│   └── EventFlowDemo.java
└── experiment06_profiles/
    └── ProfileSwitchDemo.java
```

---

## Checklist — 6 câu hỏi

Ví dụ với `@Transactional` proxy:

| # | Câu hỏi | Trả lời |
|---|---------|---------|
| 1 | Là gì? | AOP proxy wrapping method trong transaction boundary |
| 2 | Giải quyết gì? | Tự động begin/commit/rollback — không cần manual transaction management |
| 3 | Hoạt động thế nào? | BeanPostProcessor tạo proxy; proxy intercept call; PlatformTransactionManager manage |
| 4 | Khi nào dùng? | Service layer methods thay đổi data |
| 5 | Khi nào không? | Read-only → `readOnly=true`; không phải mọi method đều cần |
| 6 | Debug thế nào? | Self-invocation → transaction không active; LOG `org.springframework.transaction=DEBUG` |

---

## Run

```bash
mvn test -pl phase-09-spring-core
```
