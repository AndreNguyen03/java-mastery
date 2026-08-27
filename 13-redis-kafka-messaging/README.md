# Phase 13 — Redis, Kafka & Messaging

> **Mục tiêu:** Hiểu tại sao Redis và Kafka tồn tại, khi nào dùng, và cách debug khi hệ thống có vấn đề liên quan đến cache và message queue.

---

## Part 1 — Redis

### Redis là gì
In-memory data store — key-value nhưng với nhiều data structures.

**Tại sao không chỉ dùng DB:**
- DB access: ~5-10ms (network + disk)
- Redis access: ~0.1ms (in-memory)
- Tradeoff: không persistent by default (cấu hình được)

### Data Structures
| Structure | Commands | Dùng khi |
|-----------|---------|---------|
| String | GET, SET, INCR, EXPIRE | Cache value, counter, rate limiting |
| Hash | HGET, HSET, HMGET | Object fields (user profile) |
| List | LPUSH, RPUSH, LPOP, LRANGE | Queue, timeline, history |
| Set | SADD, SMEMBERS, SISMEMBER | Unique members, tags |
| Sorted Set | ZADD, ZRANGE, ZRANK | Leaderboard, priority queue, time-series |
| Stream | XADD, XREAD | Event log, message queue (nhẹ hơn Kafka) |

### Caching Patterns
```
Cache Aside (Lazy Loading):
  1. Check cache
  2. If miss → fetch from DB → store in cache
  3. Return

Write Through:
  1. Write to DB
  2. Write to cache
  3. Return

Write Behind (Write Back):
  1. Write to cache
  2. Return (async write to DB later)
  ↑ Risk: data loss nếu cache crash trước khi flush
```

### Spring Cache với Redis
```java
@Configuration
@EnableCaching
class CacheConfig {
    @Bean
    RedisCacheConfiguration defaultCacheConfig() {
        return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .disableCachingNullValues()
            .serializeValuesWith(SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));
    }
}

@Service
class UserService {
    @Cacheable(value = "users", key = "#id")
    public UserResponse findById(Long id) { ... }

    @CacheEvict(value = "users", key = "#id")
    public void updateUser(Long id, UpdateRequest req) { ... }

    @CachePut(value = "users", key = "#result.id()")
    public UserResponse createUser(CreateRequest req) { ... }
}
```

### Distributed Lock
```java
// Redisson hoặc Redis SET NX EX
String lockKey = "lock:order:" + orderId;
String lockValue = UUID.randomUUID().toString();

// Acquire (atomic SET if not exists, expires in 30s)
Boolean acquired = redis.opsForValue()
    .setIfAbsent(lockKey, lockValue, Duration.ofSeconds(30));

if (Boolean.TRUE.equals(acquired)) {
    try {
        processOrder(orderId);
    } finally {
        // Release chỉ nếu vẫn là lock của mình (check + delete phải atomic)
        if (lockValue.equals(redis.opsForValue().get(lockKey))) {
            redis.delete(lockKey);
        }
    }
}
```

### Rate Limiting
```java
// Sliding window counter
String key = "rate:user:" + userId;
Long count = redis.opsForValue().increment(key);
if (count == 1) redis.expire(key, Duration.ofMinutes(1));
if (count > 100) throw new RateLimitExceededException();
```

### Cache Stampede (Thundering Herd)
```
Cache expires → many requests simultaneously → all hit DB → DB overload

Solutions:
1. Probabilistic early expiration
2. Lock: only first request fetches, others wait
3. Background refresh trước khi expire
```

---

## Part 2 — Kafka

### Tại sao Kafka tồn tại
```
Synchronous call:
  OrderService → PaymentService.charge() → [wait] → done
  Problem: tight coupling, payment down → order down

Kafka:
  OrderService → Kafka (order.created) → done (không chờ)
  PaymentService ← Kafka ← consume order.created → charge async
  
Benefits: loose coupling, backpressure, replay, audit log
```

### Core Concepts
```
Topic: logical channel (như database table cho events)
Partition: physical shard của topic (parallel consumption)
Offset: position của message trong partition
Consumer Group: nhiều instances cùng group → mỗi partition được 1 instance consume
Replication: mỗi partition có N replicas (fault tolerance)
```

### Message Delivery Semantics
| Semantic | Đặc điểm | Dùng khi |
|---------|---------|---------|
| At most once | Có thể mất message | Metrics, analytics |
| At least once | Có thể duplicate | Hầu hết business events + idempotency |
| Exactly once | Không mất, không duplicate | Financial transactions (phức tạp) |

### Spring Kafka
```java
// Producer
@Service
class OrderEventPublisher {
    @Autowired KafkaTemplate<String, OrderEvent> kafkaTemplate;

    void publish(OrderCreatedEvent event) {
        kafkaTemplate.send("orders.created",
            event.orderId().toString(),  // key → same key → same partition (ordering)
            event);
    }
}

// Consumer
@Component
class PaymentConsumer {
    @KafkaListener(topics = "orders.created", groupId = "payment-service")
    void handleOrderCreated(OrderCreatedEvent event) {
        paymentService.processPayment(event.orderId());
    }
}
```

### Idempotency
```java
// Consumer phải idempotent — message có thể đến 2 lần
@KafkaListener(topics = "orders.created")
@Transactional
void handle(OrderCreatedEvent event) {
    if (paymentRepo.existsByOrderId(event.orderId())) {
        return; // already processed
    }
    paymentRepo.save(Payment.from(event));
}
```

### Dead Letter Queue (DLQ)
```java
@KafkaListener(topics = "orders.created")
void handle(OrderCreatedEvent event) {
    try {
        processOrder(event);
    } catch (Exception e) {
        // Spring Kafka tự động route sang DLQ sau N retries
        throw e;
    }
}

// Config: retry 3 times, then send to orders.created.DLT
@Bean
DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> template) {
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(template);
    return new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 3));
}
```

### Outbox Pattern
```
Vấn đề: save to DB và publish to Kafka → 2 operations → không atomic
        DB committed nhưng Kafka publish fail → data inconsistency

Solution: Outbox
  1. Save entity + Outbox event trong cùng 1 DB transaction
  2. Separate poller/CDC reads outbox table → publish to Kafka
  3. Delete from outbox sau khi published
```

---

## Project — Order Processing System

```
Architecture:
  OrderController → Kafka (order.created)
       ↗                    ↘
     Client           PaymentConsumer → payment.completed
                      InventoryConsumer → inventory.reserved
                      NotificationConsumer → email sent

Docker Compose:
  - PostgreSQL (orders)
  - Redis (cache + rate limiting)
  - Kafka + Zookeeper
  - Kafka UI (optional)
```

```
src/
├── order/
│   ├── OrderController.java
│   ├── OrderService.java          ← save + publish event
│   ├── Order.java                 ← @Entity
│   └── OutboxEvent.java           ← @Entity (outbox pattern)
├── payment/
│   ├── PaymentConsumer.java       ← consume order.created
│   ├── PaymentService.java
│   └── Payment.java
├── inventory/
│   └── InventoryConsumer.java
├── notification/
│   └── NotificationConsumer.java
├── kafka/
│   ├── KafkaConfig.java
│   ├── OrderEventPublisher.java
│   └── OutboxPoller.java          ← @Scheduled poll outbox table
├── cache/
│   └── ProductCacheService.java   ← Redis cache-aside
└── docker-compose.yml
```

**Scenarios:**
1. Normal happy path: order → payment → inventory → notification
2. Payment failure → retry → DLQ
3. Duplicate message → idempotency check
4. Redis cache miss → DB fallback → populate cache
5. Cache stampede → distributed lock
6. Rate limiting per user

---

## Checklist — 6 câu hỏi

Ví dụ với `Kafka Consumer Group`:

| # | Câu hỏi | Trả lời |
|---|---------|---------|
| 1 | Là gì? | Logical group of consumers sharing partitions of a topic |
| 2 | Giải quyết gì? | Scale consumption: N consumers trong group, mỗi partition có đúng 1 consumer |
| 3 | Hoạt động thế nào? | Broker assign partitions → rebalance khi consumer join/leave |
| 4 | Khi nào tăng instances? | Khi consumer lag tăng, nhưng max = số partitions |
| 5 | Khi nào không nên? | Nếu message cần ordering globally — 1 partition + 1 consumer |
| 6 | Debug thế nào? | `kafka-consumer-groups.sh --describe` → xem lag; nếu lag tăng → consumer chậm hoặc thiếu instances |

---

## Run

```bash
docker compose up -d
mvn spring-boot:run -pl phase-13-redis-kafka-messaging
mvn test -pl phase-13-redis-kafka-messaging
```
