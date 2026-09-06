# Phase 14 — Microservices & Distributed Systems · Lý thuyết & Lab

---

## 1. Lý thuyết

### 1.1 Monolith vs Microservices — Khi nào chuyển

**Monolith không phải kẻ thù:**
- Simple deployment (1 artifact)
- Easy debugging (single process, stack trace complete)
- No network overhead
- ACID transactions natively

**Microservices trade-offs:**
| Benefit | Cost |
|---|---|
| Independent deployment | Distributed system complexity |
| Technology diversity | Network latency + failures |
| Team autonomy | Data consistency challenges |
| Granular scaling | Operational overhead (K8s, monitoring) |
| Fault isolation | Distributed tracing |

**Khi nào chuyển sang Microservices:**
- Team lớn (> 20 devs) và deployment conflict thường xuyên
- Cần scale từng component riêng biệt (product search vs checkout)
- Khác nhau về SLA (catalog: 99.9%, payment: 99.99%)
- Domain rõ ràng (Bounded Contexts trong DDD)

**KHÔNG chuyển khi:**
- Team nhỏ (< 10 devs)
- Domain chưa ổn định (rewrite cost very high)
- Startup còn tìm product-market fit

---

### 1.2 Synchronous vs Asynchronous Communication

**Synchronous (HTTP/gRPC):**
- Request-Response: caller blocks và chờ kết quả
- Tight coupling: nếu service B down → service A fail
- Dùng cho: real-time queries, cần kết quả ngay, user-facing requests

**Asynchronous (Kafka/RabbitMQ):**
- Fire-and-forget: caller publish event, không chờ
- Loose coupling: service B có thể down, message được retry
- Dùng cho: background processing, long operations, fan-out

**Saga Pattern — Distributed Transactions:**
Microservices không dùng XA transactions (2PC) vì: network partitions, performance, availability. Thay vào đó dùng **Saga** — chuỗi local transactions với compensating transactions (rollback).

```
Choreography Saga (event-driven):
OrderService: ORDER_CREATED event
    → PaymentService listens → PAYMENT_CHARGED event
    → InventoryService listens → INVENTORY_RESERVED event
    → NotificationService listens → NOTIFICATION_SENT
    
If InventoryService fails: INVENTORY_FAILED event
    → PaymentService listens → PAYMENT_REFUNDED
    → OrderService listens → ORDER_CANCELLED

Orchestration Saga (centralized):
OrderSaga (orchestrator) coordinates:
1. Call PaymentService → success
2. Call InventoryService → fail
3. Call PaymentService.refund → compensate
4. Mark order FAILED
```

**Choreography vs Orchestration:**
| | Choreography | Orchestration |
|---|---|---|
| Complexity | Distributed → hard to trace | Centralized → easy to see |
| Coupling | Services coupled via events | Services coupled to orchestrator |
| Testing | Hard (need all services) | Easier (unit test orchestrator) |

---

### 1.3 Circuit Breaker — Fail Fast Pattern

**Problem với cascade failures:**
Service A calls B, B calls C. C is slow (overloaded). B's threads queue waiting for C. B runs out of threads. A can't get response from B. A runs out of threads. → Entire system down from C being slow.

**Circuit Breaker State Machine:**
```
                   failure rate > threshold
CLOSED (normal) ─────────────────────────→ OPEN (fail fast)
    ↑                                            │
    │                            wait-duration   │
    │          success          ↓               │
HALF-OPEN ←──────────────────────              │
(probe)   ←─────────────────────────────────────┘
                          after wait-duration
```

- **CLOSED:** Normal operation. Count failures.
- **OPEN:** Fail immediately (no actual call). Return fallback. Prevent cascading.
- **HALF-OPEN:** Allow limited test calls. If success → CLOSED. If fail → OPEN again.

**Resilience4j configuration:**
```yaml
resilience4j:
  circuitbreaker:
    instances:
      payment-service:
        sliding-window-size: 10        # last 10 calls
        failure-rate-threshold: 50     # if > 50% fail → OPEN
        wait-duration-in-open-state: 10s  # stay OPEN for 10s
        permitted-number-of-calls-in-half-open-state: 3  # test 3 calls in HALF-OPEN
```

---

### 1.35 CQRS — Command Query Responsibility Segregation

**Vấn đề với traditional CRUD:**
- Read và write share cùng model → model trở nên complex (phải thỏa mãn cả 2 use cases)
- Write cần validation, business logic, concurrency control
- Read cần joins, aggregates, denormalized views — khác hoàn toàn với write model
- Scale: read thường nhiều hơn write 10-100x → cần scale riêng

**CQRS tách thành 2 models:**

```
Write side (Commands):                  Read side (Queries):
────────────────────                    ──────────────────────
PlaceOrderCommand                       OrderSummaryView
CancelOrderCommand                      CustomerOrderHistoryView
UpdateShippingCommand                   DashboardMetricsView

→ Normalized, validates business rules  → Denormalized, fast reads, joins pre-computed
→ Event sourced hoặc CRUD               → Often separate read DB (read replica, Redis, Elasticsearch)
```

```java
// Command side: rich domain model
@Service
public class OrderCommandService {
    
    // Command: một intention với validation
    public record PlaceOrderCommand(String customerId, List<OrderItem> items, String shippingAddress) {}
    
    public OrderId placeOrder(PlaceOrderCommand cmd) {
        // Business rules on write model
        Customer customer = customerRepo.findById(cmd.customerId())
            .orElseThrow(() -> new CustomerNotFoundException(cmd.customerId()));
        
        if (!customer.isEligibleForOrders()) {
            throw new CustomerNotEligibleException("Account suspended");
        }
        
        Order order = Order.create(customer, cmd.items(), cmd.shippingAddress()); // aggregate
        orderRepo.save(order);
        
        // Publish event → read side updates its view
        eventPublisher.publish(new OrderPlacedEvent(order.getId(), order.getTotal()));
        return order.getId();
    }
}

// Query side: optimized read model
@Service
public class OrderQueryService {
    
    // Denormalized view — pre-joined data for fast reads
    public record OrderSummaryView(
        String orderId, String customerName, String customerEmail,
        BigDecimal total, String status, LocalDateTime createdAt,
        List<String> productNames // already joined
    ) {}
    
    // Can use different data store — Elasticsearch, Redis, read replica
    public List<OrderSummaryView> getCustomerOrders(String customerId) {
        return orderReadRepository.findByCustomerId(customerId); // hits read DB
    }
    
    public OrderSummaryView getOrderDetails(String orderId) {
        return orderReadRepository.findById(orderId);
    }
}

// Event listener updates read model
@Component
public class OrderReadModelUpdater {
    @EventListener
    public void on(OrderPlacedEvent event) {
        // Denormalize: join order + customer + products → store in read DB
        Order order = orderRepo.findById(event.orderId());
        Customer customer = customerRepo.findById(order.getCustomerId());
        
        OrderSummaryView view = new OrderSummaryView(
            order.getId(), customer.getName(), customer.getEmail(),
            order.getTotal(), order.getStatus().name(), order.getCreatedAt(),
            order.getItems().stream().map(i -> i.getProductName()).toList()
        );
        orderReadRepository.save(view);
    }
}
```

**Eventual consistency:** Sau khi write, read model chưa update ngay → **eventually consistent**. Acceptable cho most use cases (user thấy "đang xử lý" trong vài milliseconds). Không acceptable cho payment balance.

**Khi nào dùng CQRS:**
- Read/write patterns rất khác nhau
- Cần scale read và write độc lập
- Complex reporting/analytics queries
- **KHÔNG dùng cho:** simple CRUD apps — added complexity không worth it

---

### 1.36 Event Sourcing

**Traditional state storage:** Lưu current state. `UPDATE orders SET status='SHIPPED'` → quá khứ bị xóa.

**Event Sourcing:** Lưu **tất cả sự kiện đã xảy ra** thay vì current state. Current state = apply all events.

```java
// Events — immutable facts
public sealed interface OrderEvent permits
    OrderPlaced, OrderPaid, OrderShipped, OrderCancelled {}

record OrderPlaced(String orderId, String customerId, List<OrderItem> items,
                   BigDecimal total, Instant occurredAt) implements OrderEvent {}
record OrderPaid(String orderId, String paymentId, Instant occurredAt) implements OrderEvent {}
record OrderShipped(String orderId, String trackingNumber, Instant occurredAt) implements OrderEvent {}
record OrderCancelled(String orderId, String reason, Instant occurredAt) implements OrderEvent {}

// Aggregate: state derived by replaying events
class Order {
    private String id;
    private String customerId;
    private OrderStatus status;
    private String paymentId;
    private String trackingNumber;
    
    // Reconstruct current state from event history
    public static Order reconstitute(List<OrderEvent> events) {
        Order order = new Order();
        for (OrderEvent event : events) {
            order.apply(event); // each event mutates state
        }
        return order;
    }
    
    private void apply(OrderEvent event) {
        switch (event) {
            case OrderPlaced e -> {
                this.id = e.orderId();
                this.customerId = e.customerId();
                this.status = OrderStatus.PENDING;
            }
            case OrderPaid e -> {
                this.paymentId = e.paymentId();
                this.status = OrderStatus.PAID;
            }
            case OrderShipped e -> {
                this.trackingNumber = e.trackingNumber();
                this.status = OrderStatus.SHIPPED;
            }
            case OrderCancelled e -> this.status = OrderStatus.CANCELLED;
        }
    }
}

// Event Store — append-only log
interface EventStore {
    void append(String aggregateId, List<OrderEvent> newEvents, long expectedVersion);
    List<OrderEvent> loadEvents(String aggregateId);
    List<OrderEvent> loadEvents(String aggregateId, long fromVersion);
}
```

**Benefits:**
1. **Complete audit log** — mọi thay đổi được ghi lại với timestamp → compliance, debugging
2. **Time travel** — reconstruct state tại bất kỳ điểm nào trong quá khứ
3. **Event replay** — rebuild read models bằng cách replay events
4. **Natural fit cho CQRS** — events trigger read model updates

**Challenges:**
- **Eventual consistency** trong read model
- **Schema evolution** — old events phải vẫn readable sau khi schema thay đổi (versioning)
- **Snapshots** cần thiết khi event history quá dài (mỗi lần load phải replay 10,000 events)
- Query current state không trivial — cần read model

```java
// Snapshot optimization: snapshot every N events
public Order loadWithSnapshot(String orderId) {
    Optional<Snapshot> snapshot = snapshotStore.findLatest(orderId);
    
    if (snapshot.isPresent()) {
        List<OrderEvent> recentEvents = eventStore.loadEvents(
            orderId, snapshot.get().version() + 1 // only events after snapshot
        );
        return snapshot.get().restoreOrder().applyAll(recentEvents);
    }
    
    return Order.reconstitute(eventStore.loadEvents(orderId));
}
```

---

### 1.37 Bulkhead Pattern

**Vấn đề:** Service A gọi Service B và Service C. Service B bị chậm → tất cả threads của A bị occupied chờ B → không còn thread nào xử lý requests tới C → **C bị ảnh hưởng dù C hoàn toàn healthy**.

**Bulkhead** (vách ngăn tàu): Cách ly resource pools để failure của một phần không drag cả system.

```java
// Dedicated thread pool per downstream service
@Configuration
public class ThreadPoolConfig {
    
    // Payment service có thread pool riêng — max 10 threads
    @Bean("paymentExecutor")
    public Executor paymentExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(25);
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setThreadNamePrefix("payment-");
        executor.initialize();
        return executor;
    }
    
    // Inventory service có thread pool riêng — max 5 threads
    @Bean("inventoryExecutor")
    public Executor inventoryExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(3);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(10);
        executor.setThreadNamePrefix("inventory-");
        executor.initialize();
        return executor;
    }
}

// Nếu payment service slow → chỉ payment threads bị block
// Inventory service vẫn dùng inventory thread pool → không bị ảnh hưởng
@Service
public class OrderService {
    @Async("paymentExecutor") // uses dedicated pool
    public CompletableFuture<PaymentResult> processPayment(PaymentRequest req) {
        return CompletableFuture.supplyAsync(() -> paymentClient.charge(req));
    }
    
    @Async("inventoryExecutor") // uses separate pool — isolated
    public CompletableFuture<InventoryResult> reserveInventory(ReserveRequest req) {
        return CompletableFuture.supplyAsync(() -> inventoryClient.reserve(req));
    }
}
```

**Resilience4j Bulkhead:**
```yaml
resilience4j:
  bulkhead:
    instances:
      payment-service:
        max-concurrent-calls: 10  # max concurrent calls to payment service
        max-wait-duration: 0ms    # don't queue — fail immediately if full
      inventory-service:
        max-concurrent-calls: 5
```

```java
@Bulkhead(name = "payment-service", type = Bulkhead.Type.SEMAPHORE,
          fallbackMethod = "paymentFallback")
public PaymentResult charge(PaymentRequest req) {
    return paymentClient.charge(req);
}

PaymentResult paymentFallback(PaymentRequest req, BulkheadFullException e) {
    // Payment bulkhead full — too many concurrent requests
    throw new ServiceOverloadedException("Payment service is busy");
}
```

**Bulkhead + Circuit Breaker kết hợp:**
- **Bulkhead:** Giới hạn concurrent requests → prevent thread exhaustion
- **Circuit Breaker:** Fail fast khi failure rate cao → prevent cascade
- **Timeout:** Giới hạn waiting time → prevent indefinite blocking
- Dùng cả 3 cùng nhau: `@Bulkhead + @CircuitBreaker + @TimeLimiter`

---

### 1.4 API Gateway Pattern

**Tại sao cần API Gateway:**
- Clients biết về tất cả microservices URLs → coupling
- Cross-cutting concerns (auth, rate limiting, logging) duplicate in every service
- CORS, SSL termination duplicate

**API Gateway responsibilities:**
1. **Routing:** `/api/orders/**` → Order Service
2. **Authentication:** Verify JWT before routing
3. **Rate limiting:** Throttle per client
4. **Circuit breaker:** If downstream down → fallback
5. **Aggregation:** Combine responses from multiple services (BFF pattern)
6. **Observability:** Centralized logging, tracing

---

### 1.5 Distributed Tracing — Debugging Across Services

**Problem:** Request fails. Error log trong Payment Service. But request went: Client → Gateway → Order Service → Payment Service → Inventory Service. Which service caused the issue?

**Tracing solution (Zipkin + Micrometer):**
- Each request gets a **TraceId** (same across all services)
- Each span represents one operation (single service call, DB query)
- **SpanId:** Unique per operation

```
Client → Gateway (traceId=abc, spanId=1)
           → Order Service (traceId=abc, spanId=2, parentSpanId=1)
               → Payment Service (traceId=abc, spanId=3, parentSpanId=2)
               → Inventory Service (traceId=abc, spanId=4, parentSpanId=2)
```

Trace propagated via HTTP headers:
- `X-B3-TraceId: abc`
- `X-B3-SpanId: 2`
- `X-B3-ParentSpanId: 1`

Zipkin UI shows waterfall diagram of entire request path.

---

### 1.6 CAP Theorem

**Trong distributed system, chỉ đảm bảo được 2 trong 3:**
- **C (Consistency):** Mọi node thấy cùng data tại cùng thời điểm
- **A (Availability):** Mọi request nhận response (success hoặc error, không timeout)
- **P (Partition Tolerance):** System tiếp tục hoạt động khi network partition xảy ra

**Network partition luôn xảy ra → phải chọn C hoặc A:**
- **CP (Consistency + Partition Tolerance):** Khi partition → từ chối requests đến ensure consistent data. Ví dụ: HBase, Zookeeper
- **AP (Availability + Partition Tolerance):** Khi partition → trả stale data nhưng remain available. Ví dụ: Cassandra, DynamoDB, Redis (cluster mode)

**Practical implications:**
- Banking: CP — không thể hiển thị sai số dư
- Social feed: AP — ok nếu feed hơi stale vài giây

---

## 2. Vấn đề thường gặp & Cách fix

### Issue 1: Saga Compensating Transaction Fail

```
Order Saga:
1. ✅ Create order
2. ✅ Charge payment (paymentId=P123)
3. ❌ Reserve inventory FAIL
4. 🔄 Compensation: Refund payment P123
5. ❌ REFUND ALSO FAILS!
```

**Fix:** Compensation failures must be handled separately:
- Alert operations team immediately
- Store compensation state in DB — retry periodically
- Idempotent compensations — safe to retry
- Manual intervention workflow

### Issue 2: Feign client timeout không được set

```java
// ❌ Default: Feign waits forever
@FeignClient(name = "product-service")
interface ProductClient {
    @GetMapping("/products/{id}")
    Product getProduct(@PathVariable String id);
}

// ✅ Set timeouts via properties
feign:
  client:
    config:
      product-service:
        connect-timeout: 2000  # 2 seconds
        read-timeout: 5000     # 5 seconds
```

### Issue 3: Circuit breaker không open

**Nguyên nhân:** Timeout chưa được counted như failure.

```yaml
# Explicitly include timeouts as failure
resilience4j:
  circuitbreaker:
    instances:
      payment-service:
        record-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
          - feign.FeignException
```

---

## 3. Code mẫu

```yaml
# api-gateway/application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: http://localhost:8083
          predicates:
            - Path=/api/v1/orders/**
          filters:
            - name: CircuitBreaker
              args:
                name: orderServiceCB
                fallbackUri: forward:/fallback/orders
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200

resilience4j:
  circuitbreaker:
    instances:
      orderServiceCB:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
```

```java
// saga/OrderSaga.java
@Component
public class OrderSaga {

    private final OrderRepository orderRepository;
    private final PaymentClient paymentClient;
    private final InventoryClient inventoryClient;

    /**
     * Orchestration Saga: centralized coordination.
     * Each step is a local transaction. On failure: compensate.
     * 
     * Why not distributed transactions (XA)?
     * - Require all participants to support 2PC (Kafka doesn't)
     * - Network partition during 2PC → system stuck
     * - Very long lock times → poor performance
     */
    public Order execute(CreateOrderRequest req) {
        Order order = orderRepository.save(new Order(req, OrderStatus.PENDING));
        log.info("Saga step 1 complete: orderId={} created", order.getId());

        // Step 2: Charge payment
        String paymentId = null;
        try {
            paymentId = paymentClient.charge(
                new ChargeRequest(order.getId(), req.userId(), req.totalAmount())
            ).paymentId();
            log.info("Saga step 2 complete: paymentId={} charged", paymentId);
        } catch (Exception e) {
            log.error("Saga step 2 failed: payment failed for orderId={}", order.getId(), e);
            // Compensation: cancel order (no payment yet)
            order.cancel("Payment failed: " + e.getMessage());
            orderRepository.save(order);
            throw new SagaException("Order saga failed at payment", e);
        }

        // Step 3: Reserve inventory
        try {
            inventoryClient.reserve(
                new ReserveRequest(order.getId(), req.productId(), req.quantity())
            );
            log.info("Saga step 3 complete: inventory reserved for orderId={}", order.getId());
        } catch (Exception e) {
            log.error("Saga step 3 failed: inventory failed for orderId={}", order.getId(), e);
            
            // Compensation step 2: refund payment
            final String pid = paymentId;
            try {
                paymentClient.refund(new RefundRequest(pid, "Inventory unavailable"));
                log.info("Compensation: payment {} refunded", pid);
            } catch (Exception refundEx) {
                // Compensation itself failed! This needs human intervention.
                log.error("CRITICAL: Compensation failed! paymentId={} needs manual refund. Error: {}",
                    pid, refundEx.getMessage(), refundEx);
                alertOpsTeam(pid, order.getId(), "Refund failed - manual action required");
            }
            
            // Compensation step 1: cancel order
            order.cancel("Inventory unavailable");
            orderRepository.save(order);
            throw new SagaException("Order saga failed at inventory", e);
        }

        order.confirm(paymentId);
        return orderRepository.save(order);
    }
}
```

```java
// client/PaymentClientWithCB.java
@Component
public class PaymentClientWithCB {

    private final PaymentClient paymentClient;
    private static final Logger log = LoggerFactory.getLogger(PaymentClientWithCB.class);

    public PaymentClientWithCB(PaymentClient paymentClient) {
        this.paymentClient = paymentClient;
    }

    /**
     * Stacked annotations — applied in order: TimeLimiter → Retry → CircuitBreaker
     * 1. TimeLimiter: timeout if takes > 3s
     * 2. Retry: retry up to 3 times with exponential backoff
     * 3. CircuitBreaker: if too many failures → open circuit
     */
    @CircuitBreaker(name = "payment-service", fallbackMethod = "chargeFallback")
    @Retry(name = "payment-service")
    @TimeLimiter(name = "payment-service")
    public CompletableFuture<ChargeResponse> charge(ChargeRequest request) {
        return CompletableFuture.supplyAsync(() -> paymentClient.charge(request));
    }

    // Fallback when circuit is OPEN — must match signature + Throwable
    CompletableFuture<ChargeResponse> chargeFallback(ChargeRequest request, Throwable ex) {
        log.warn("Payment CB open for orderId={}: {}", request.orderId(), ex.getMessage());
        return CompletableFuture.failedFuture(
            new ServiceUnavailableException("Payment service temporarily unavailable")
        );
    }
}
```

```java
// tracing/TracingConfig.java
@Configuration
public class TracingConfig {

    @Bean
    public Sampler defaultSampler() {
        return Sampler.ALWAYS_SAMPLE; // 100% in dev, use 0.1f in prod
    }
}

// Manual span creation for custom operations
@Service
public class OrderService {
    @Autowired private Tracer tracer;

    public void processWithTrace(String orderId) {
        Span span = tracer.nextSpan().name("order.process");
        try (Tracer.SpanInScope scope = tracer.withSpan(span.start())) {
            span.tag("orderId", orderId);
            span.tag("operation", "process");
            
            doProcessing(orderId);
            
            span.tag("result", "success");
        } catch (Exception ex) {
            span.error(ex); // marks span as error in Zipkin
            throw ex;
        } finally {
            span.end(); // always end the span
        }
    }
}
```

```java
// idempotency/IdempotencyFilter.java
@Component
public class IdempotencyFilter extends OncePerRequestFilter {

    private final RedisTemplate<String, String> redis;

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String key = req.getHeader("Idempotency-Key");

        if (key != null && isMutatingMethod(req.getMethod())) {
            String cacheKey = "idempotency:" + key;
            String cached = redis.opsForValue().get(cacheKey);

            if (cached != null) {
                // Return exact same response as first request
                res.setContentType(MediaType.APPLICATION_JSON_VALUE);
                res.setStatus(200);
                res.getWriter().write(cached);
                return;
            }

            // Wrap response to capture it
            ContentCachingResponseWrapper wrapper = new ContentCachingResponseWrapper(res);
            chain.doFilter(req, wrapper);

            // Cache response for 24 hours
            String body = new String(wrapper.getContentAsByteArray(), StandardCharsets.UTF_8);
            redis.opsForValue().set(cacheKey, body, Duration.ofHours(24));
            wrapper.copyBodyToResponse();
            return;
        }

        chain.doFilter(req, res);
    }

    private boolean isMutatingMethod(String method) {
        return "POST".equals(method) || "PUT".equals(method) || "PATCH".equals(method);
    }
}
```

---

## 4. Lab Steps

1. **Happy path:** Create order through API Gateway → trace in Zipkin shows all 3 services
2. **Circuit breaker:** Stop payment-service → send 10 requests → CB OPEN → check Actuator: `/actuator/circuitbreakerevents`
3. **CB half-open:** Restart payment-service → wait 10s → send request → CB goes HALF-OPEN → CLOSED
4. **Saga compensation:** Force inventory service to fail → observe payment refund log
5. **Idempotency:** Same request with `Idempotency-Key: test-123` twice → DB has 1 record
6. **Consumer lag:** Stop payment consumer → produce 100 orders → restart consumer → observe catch-up

---

## 5. Checklist tự kiểm tra

- [ ] Circuit Breaker state transitions: CLOSED → OPEN → HALF-OPEN → CLOSED/OPEN
- [ ] Saga: step 3 fail → step 2 compensated (refund) → order cancelled
- [ ] Distributed trace: same traceId appears in ALL service logs for single request
- [ ] Idempotency key: second request returns cached response, no DB write
- [ ] Services có thể chạy độc lập — service B down không ảnh hưởng service A startup
- [ ] API Gateway: authentication check tại gateway, services trust gateway's header
- [ ] CAP: hiểu tại sao không thể có cả 3 trong distributed system
