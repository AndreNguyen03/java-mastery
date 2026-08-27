# Phase 14 — Microservices & Distributed System

> **Mục tiêu:** Hiểu tại sao distributed systems khó, biết các pattern giải quyết vấn đề gì, và đưa ra trade-off decision có lý do.

---

## Part 1 — Monolith → Microservices

### Khi nào KHÔNG nên microservices
```
Bạn không cần microservices nếu:
- Team nhỏ (< 10 người)
- Domain chưa rõ ràng
- Throughput thấp
- Latency không phải vấn đề

Monolith first → Extract khi có pain point rõ ràng
```

### Modular Monolith (trung gian)
```
1 deployment unit, nhưng code tổ chức như services
Module User, Order, Payment... communicate qua internal interfaces
→ Dễ extract thành microservice sau
```

### Khi nào nên microservices
- Cần scale từng service độc lập (payment cần 10x machines hơn notification)
- Team lớn, mỗi team own một service
- Different tech requirements (Python ML service + Java API)
- Compliance (payment service cần PCI-DSS isolation)

---

## Part 2 — Service Communication

### Synchronous (REST / gRPC)
```
API Gateway
    ↓
Order Service ─────→ Product Service (REST/Feign)
    └──────────────→ Inventory Service
```

```java
// OpenFeign declarative HTTP client
@FeignClient(name = "product-service", url = "${services.product.url}")
interface ProductClient {
    @GetMapping("/api/v1/products/{id}")
    ProductResponse findById(@PathVariable Long id);
}
```

### Asynchronous (Event-driven)
```
Order Service → Kafka (order.created)
                   ├── Payment Service (consume)
                   ├── Inventory Service (consume)
                   └── Notification Service (consume)
```

### Sync vs Async
| | Sync | Async |
|--|------|-------|
| Coupling | Tight | Loose |
| Latency | Low (direct) | Higher (eventual) |
| Availability | Dependent on downstream | Independent |
| Complexity | Simple | Complex (ordering, idempotency) |
| Dùng khi | Query (need answer now) | Command (fire and forget) |

---

## Part 3 — Resilience

### Circuit Breaker (Resilience4j)
```
CLOSED (normal)
  → success rate drops below threshold
OPEN (fail fast, không gọi downstream)
  → after timeout
HALF_OPEN (test một số requests)
  → nếu success → CLOSED; nếu fail → OPEN lại
```

```java
@CircuitBreaker(name = "product-service", fallbackMethod = "getProductFallback")
ProductResponse getProduct(Long id) {
    return productClient.findById(id);
}

ProductResponse getProductFallback(Long id, Exception ex) {
    return ProductResponse.unknown(id); // cached or default response
}
```

### Retry với Exponential Backoff
```java
@Retry(name = "product-service")
ProductResponse getProduct(Long id) { ... }

// application.yml
resilience4j:
  retry:
    instances:
      product-service:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        # 500ms → 1000ms → 2000ms
```

### Bulkhead
```java
// Giới hạn concurrent calls đến một service
@Bulkhead(name = "product-service", type = Bulkhead.Type.THREADPOOL)
CompletableFuture<ProductResponse> getProductAsync(Long id) { ... }
```

### Timeout
```java
@TimeLimiter(name = "product-service")
CompletableFuture<ProductResponse> getProduct(Long id) { ... }
```

---

## Part 4 — Distributed Transaction

### Vấn đề
```
// 2-phase commit (2PC) — không dùng trong microservices
// Network partition → coordinator crash → stuck

// SAGA pattern thay thế
```

### Saga Pattern

**Choreography (event-driven):**
```
OrderService.createOrder()
    → emit order.created
        → PaymentService.charge()
        → emit payment.completed
            → InventoryService.reserve()
            → emit inventory.reserved
                → order.confirmed

Compensating transactions (rollback):
inventory.reservation.failed
    → PaymentService.refund()
    → emit payment.refunded
        → OrderService.cancelOrder()
```

**Orchestration (central coordinator):**
```
OrderSaga orchestrates:
  1. Call PaymentService
  2. If success → call InventoryService
  3. If fail → call PaymentService.refund()
  4. If success → complete order
```

| | Choreography | Orchestration |
|--|-------------|---------------|
| Coupling | Loose | Tighter (saga knows all steps) |
| Visibility | Hard to track | Easy to monitor |
| Complexity | Distributed logic | Centralized logic |

---

## Part 5 — CQRS & Event Sourcing

### CQRS (Command Query Responsibility Segregation)
```
Write side: Command → CommandHandler → Domain → Write DB
Read side:  Query → QueryHandler → Read DB (optimized projections)

Sync via: events / CDC / scheduled jobs
```

**Khi nào:** Write và read có different performance/scaling needs.

### Event Sourcing
```
Thay vì lưu state hiện tại:
  Account { balance: 1000 }

Lưu sequence of events:
  AccountCreated { amount: 500 }
  MoneyDeposited { amount: 700 }
  MoneyWithdrawn { amount: 200 }
  → Replay → balance = 1000

Benefits: audit log, time travel, event replay
Drawbacks: complexity, eventual consistency, snapshot cần thiết
```

---

## Part 6 — Distributed Lock & Idempotency

### Distributed Lock (Redis)
```java
// Đảm bảo chỉ 1 instance xử lý một job
String lockKey = "job:invoice-generation:" + date;
if (redisLock.tryAcquire(lockKey, 30, TimeUnit.SECONDS)) {
    try { generateInvoices(date); }
    finally { redisLock.release(lockKey); }
}
```

### Idempotency Key
```
Client gửi request với Idempotency-Key header
Server lưu key + response vào Redis (TTL 24h)
Nếu duplicate request → trả về cached response
→ Safe to retry
```

---

## Part 7 — CAP Theorem

```
Consistency: mọi read nhận được write mới nhất
Availability: mọi request nhận được response (không timeout)
Partition tolerance: hệ thống tiếp tục hoạt động dù có network partition

CAP: không thể có cả 3 cùng lúc khi có partition

CP systems: PostgreSQL, HBase (choose C over A)
AP systems: Cassandra, DynamoDB (choose A over C)
```

**Trong thực tế:** Network partition xảy ra, nên phải chọn C hoặc A.

---

## Project — E-commerce Microservices

```
docker-compose.yml:
  ├── api-gateway (Spring Cloud Gateway)
  ├── user-service    (port 8081)
  ├── product-service (port 8082)
  ├── order-service   (port 8083)
  ├── payment-service (port 8084)
  ├── notification-service (port 8085)
  ├── PostgreSQL (per service schema)
  ├── Redis
  ├── Kafka
  └── Zipkin (distributed tracing)

phase-14-microservices-distributed-system/
├── api-gateway/
├── user-service/
├── product-service/
├── order-service/         ← Saga orchestrator
├── payment-service/
├── notification-service/
├── docker-compose.yml
└── concepts/              ← standalone experiments
    ├── SagaDemo.java
    ├── CircuitBreakerDemo.java
    └── DistributedLockDemo.java
```

**Flows để implement:**
1. `POST /orders` → Saga: payment + inventory → confirmed/failed
2. Circuit breaker: tắt product-service → order fallback
3. Distributed tracing: trace ID qua tất cả services
4. Idempotency: retry order → chỉ charge 1 lần

---

## Checklist — 6 câu hỏi

Ví dụ với `Circuit Breaker`:

| # | Câu hỏi | Trả lời |
|---|---------|---------|
| 1 | Là gì? | State machine: CLOSED/OPEN/HALF_OPEN — ngăn calls đến service đang lỗi |
| 2 | Giải quyết gì? | Cascading failure: upstream fail → caller thread pool exhausted → caller fail |
| 3 | Hoạt động thế nào? | Track success/failure rate; OPEN khi threshold; HALF_OPEN để test recovery |
| 4 | Khi nào dùng? | Sync calls đến external services |
| 5 | Khi nào không? | Async / event-driven — dùng DLQ và retry thay |
| 6 | Debug thế nào? | `/actuator/circuitbreakers` endpoint; OPEN state → kiểm tra downstream health |

---

## Run

```bash
docker compose up -d
# Start services individually
mvn spring-boot:run -pl phase-14-microservices-distributed-system
```
