# Phase 13 — Redis, Kafka & Messaging · Lý thuyết & Lab

---

## 1. Lý thuyết

### 1.1 Redis — Tại sao cần Cache

**Vấn đề không có cache:**
- DB query P99: 50ms. 1000 req/s → 1000 DB queries/s → DB bottleneck
- Đối với static-ish data (product catalog, config) — DB trả kết quả giống nhau mỗi lần → waste

**Cache-Aside Pattern (Lazy Loading):**
```
Read: Check cache → miss? → read DB → store in cache → return
Write: Update DB → invalidate cache (hoặc update cache)
```

**Tại sao invalidate chứ không phải update cache on write:**
Race condition: Thread A invalidates → Thread B reads old data from cache → Thread A updates DB → cache stale. Với invalidate: Thread B on miss → reads DB → always fresh data.

**Cache Stampede (Thundering Herd):**
Xảy ra khi: cache expires (TTL) → nhiều concurrent requests cùng lúc thấy cache miss → tất cả cùng hit DB.

**Fix: probabilistic early expiration** hoặc dùng distributed lock:
```java
// Chỉ 1 request được phép compute, phần còn lại chờ hoặc trả stale data
String lockKey = "lock:product:" + productId;
boolean locked = redis.setIfAbsent(lockKey, "1", Duration.ofSeconds(5));
if (locked) {
    try {
        Product product = loadFromDB(productId);
        redis.set(cacheKey, product, Duration.ofMinutes(10));
        return product;
    } finally {
        redis.delete(lockKey);
    }
} else {
    // Return stale data or wait briefly and retry
    return redis.get(cacheKey);
}
```

---

### 1.2 Redis Data Structures

Redis không chỉ là key-value store — có nhiều data structures:

| Structure | Commands | Use Case |
|---|---|---|
| String | GET, SET, INCR, EXPIRE | Simple cache, counters, rate limiting |
| List | LPUSH, RPOP, LRANGE | Queue, recent activity log |
| Hash | HSET, HGET, HGETALL | Object fields, avoid serialization |
| Set | SADD, SMEMBERS, SINTER | Tags, unique visitors, permissions |
| Sorted Set | ZADD, ZRANGE, ZRANGEBYSCORE | Leaderboard, rate limiting, delayed jobs |
| Stream | XADD, XREAD | Event log, alternative to Kafka for simple cases |

**Hash cho objects:**
```java
// vs String (serialize entire object)
redis.opsForHash().put("user:123", "name", "Alice");
redis.opsForHash().put("user:123", "email", "alice@test.com");
// Benefit: update single field without deserialize/serialize entire object
redis.opsForHash().put("user:123", "lastLogin", Instant.now().toString());
```

---

### 1.3 Kafka — Event Streaming

**Kafka vs Traditional MQ (RabbitMQ):**
| | RabbitMQ | Kafka |
|---|---|---|
| Storage | In-memory, messages deleted after ack | Persistent log, retained for days |
| Ordering | Per queue | Per partition |
| Replay | Not supported | Consumers can seek to any offset |
| Throughput | Moderate (100K msg/s) | Very high (millions msg/s) |
| Use case | Task queue, RPC | Event log, stream processing |

**Topic, Partition, Offset:**
```
Topic "orders" with 3 partitions:
Partition 0: [offset 0: order-A] [offset 1: order-D] [offset 2: order-G]
Partition 1: [offset 0: order-B] [offset 1: order-E]
Partition 2: [offset 0: order-C] [offset 1: order-F]

Consumer Group "payment-service" (3 consumers):
  Consumer 1 → Partition 0
  Consumer 2 → Partition 1
  Consumer 3 → Partition 2
```

**Key-based partitioning:** Kafka hash(key) → partition. Same key → same partition → **ordering guaranteed within a key**. Ví dụ: key=orderId → tất cả events của cùng order đến cùng partition → consumer xử lý theo thứ tự.

**Consumer Groups:** Mỗi consumer group nhận MỌI messages (logical broadcast). Trong group, partitions được phân chia giữa consumers (parallel processing). Thêm consumer vào group → rebalance → tăng throughput.

---

### 1.4 Delivery Semantics

**At-most-once:** Ack trước khi process → nếu crash → message lost. Dùng cho metrics, logs (acceptable loss).

**At-least-once:** Process trước khi ack → nếu crash after process nhưng before ack → message processed lại. **Default trong Kafka.** Consumer phải **idempotent**.

**Exactly-once:** Kafka transactions (producer + consumer in same transaction). Complex, overhead, thường không cần.

**Idempotency implementation:**
```java
// Check idempotency key before processing
@KafkaListener(topics = "orders.created")
@Transactional
public void handleOrderCreated(ConsumerRecord<String, OrderCreatedEvent> record) {
    String orderId = record.value().orderId();
    
    // Idempotency check: have we already processed this?
    if (paymentRepository.existsByOrderId(orderId)) {
        log.warn("Duplicate message for orderId={}, skipping", orderId);
        return; // idempotent: same result as first processing
    }
    
    // Process...
    paymentRepository.save(new Payment(orderId, ...));
}
```

---

### 1.5 Dead Letter Queue (DLQ)

**Tại sao cần DLQ:**
Consumer xử lý message fail → retry loop → poison pill message block entire consumer. DLQ: sau N retries → send to dead letter topic → consumer continues.

```
Producer → [orders.created] → Consumer
                                 ↓ fail
                               Retry 1 (1s)
                                 ↓ fail
                               Retry 2 (1s)
                                 ↓ fail
                               Retry 3 (1s)
                                 ↓ still fail
                             [orders.created.DLT] ← DLQ topic
                             → Monitor/Alert
                             → Manual reprocess or discard
```

---

### 1.6 Outbox Pattern — Transactional Messaging

**Vấn đề:** DB và Kafka không trong cùng transaction. Nếu save to DB thành công nhưng Kafka publish fail → data inconsistency.

**Solution: Outbox Table**
```sql
-- Tạo table "outbox" cùng DB với business tables
CREATE TABLE outbox (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    topic VARCHAR(255) NOT NULL,
    message_key VARCHAR(255),
    payload JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    sent_at TIMESTAMP -- null = not yet sent
);
```

```java
@Transactional // single transaction: save order + outbox entry
public void createOrder(Order order) {
    Order saved = orderRepo.save(order);
    // Same transaction → if either fails, both rollback
    outboxRepo.save(new OutboxEntry("orders.created", order.getId(), toJson(event)));
}

// Separate process: poll outbox table, publish to Kafka, mark sent
@Scheduled(fixedDelay = 1000)
@Transactional
public void publishOutboxMessages() {
    List<OutboxEntry> pending = outboxRepo.findBySentAtNull();
    pending.forEach(entry -> {
        kafkaTemplate.send(entry.getTopic(), entry.getMessageKey(), entry.getPayload());
        entry.setSentAt(Instant.now());
    });
}
```

---

## 2. Vấn đề thường gặp & Cách fix

### Issue 1: Kafka consumer lag tăng

**Nguyên nhân:** Consumer processing chậm hơn producer rate.

**Chẩn đoán:**
```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group payment-service
# Xem CONSUMER-ID, LAG column → nếu LAG tăng → vấn đề
```

**Fix options:**
1. Tăng `concurrency` của `KafkaListenerContainerFactory` → nhiều consumer threads
2. Tối ưu consumer processing (DB query chậm, external API call?)
3. Tăng số partitions → thêm consumer instances

### Issue 2: Cache với stale data sau write

```java
// ❌ Update DB nhưng quên evict cache
@Transactional
public Product updatePrice(Long id, BigDecimal newPrice) {
    Product product = repo.findById(id).orElseThrow();
    product.setPrice(newPrice);
    return repo.save(product);
    // Cache vẫn có giá cũ!
}

// ✅ Evict cache after update
@Transactional
@CacheEvict(value = "products", key = "#id")
public Product updatePrice(Long id, BigDecimal newPrice) {
    Product product = repo.findById(id).orElseThrow();
    product.setPrice(newPrice);
    return repo.save(product);
    // Cache entry for this id is deleted → next read will fetch from DB
}
```

### Issue 3: Rate limiter race condition

```java
// ❌ Không atomic — race condition khi concurrent requests
Long count = redis.get(key);
if (count == null) {
    redis.set(key, 1, Duration.ofMinutes(1));
} else if (count < limit) {
    redis.increment(key); // race: two threads both read count=9, both increment → 11
}

// ✅ INCR là atomic operation
Long count = redis.increment(key);
if (count == 1) redis.expire(key, Duration.ofMinutes(1)); // set TTL on first
return count <= limit;
```

---

## 3. Code mẫu

```yaml
# docker-compose.yml
services:
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0
    environment: { ZOOKEEPER_CLIENT_PORT: 2181 }

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    ports: ["9092:9092"]
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports: ["8090:8080"]
    environment:
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
```

```java
// order/OrderService.java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate;

    @Transactional
    public Order createOrder(CreateOrderRequest req) {
        Order order = orderRepository.save(new Order(req));

        OrderCreatedEvent event = new OrderCreatedEvent(order.getId(), req.userId(),
            req.amount(), req.productId(), req.quantity(), Instant.now());

        // Send with orderId as key → same order always goes to same partition → ordering
        kafkaTemplate.send("orders.created", order.getId(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) log.error("Failed to publish event for orderId={}", order.getId(), ex);
                else log.info("Event published: orderId={}, partition={}, offset={}",
                    order.getId(), result.getRecordMetadata().partition(),
                    result.getRecordMetadata().offset());
            });

        return order;
    }
}
```

```java
// payment/PaymentConsumer.java
@Component
public class PaymentConsumer {
    private final PaymentRepository paymentRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @KafkaListener(topics = "orders.created", groupId = "payment-service")
    @Transactional
    public void handleOrderCreated(ConsumerRecord<String, OrderCreatedEvent> record) {
        OrderCreatedEvent event = record.value();

        // Idempotency: at-least-once delivery → may receive same message twice
        if (paymentRepository.existsByOrderId(event.orderId())) {
            log.warn("Duplicate payment for orderId={}", event.orderId());
            return;
        }

        try {
            boolean success = processPayment(event.amount(), event.userId());
            if (success) {
                Payment payment = paymentRepository.save(new Payment(event.orderId(), event.amount()));
                kafkaTemplate.send("payments.completed",
                    new PaymentCompletedEvent(event.orderId(), payment.getId(), Instant.now()));
            } else {
                kafkaTemplate.send("payments.failed",
                    new PaymentFailedEvent(event.orderId(), "Insufficient funds", Instant.now()));
            }
        } catch (Exception e) {
            log.error("Payment processing failed for orderId={}", event.orderId(), e);
            throw e; // Rethrow → Spring Kafka retries → DLQ after max retries
        }
    }
}
```

```java
// kafka/KafkaConfig.java
@Configuration
public class KafkaConfig {

    /**
     * DefaultErrorHandler: retry 3 times with 1s delay → then DLQ
     * DLQ topic name: <original-topic>.DLT (e.g. orders.created.DLT)
     */
    @Bean
    public DefaultErrorHandler errorHandler(KafkaTemplate<Object, Object> kafkaTemplate) {
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(kafkaTemplate);
        return new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 3L));
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            ConsumerFactory<String, Object> consumerFactory,
            DefaultErrorHandler errorHandler) {
        var factory = new ConcurrentKafkaListenerContainerFactory<String, Object>();
        factory.setConsumerFactory(consumerFactory);
        factory.setCommonErrorHandler(errorHandler);
        factory.setConcurrency(3); // 3 consumer threads per @KafkaListener instance
        return factory;
    }
}
```

```java
// cache/ProductCacheService.java
@Service
public class ProductCacheService {
    private final RedisTemplate<String, Object> redisTemplate;
    private final ProductRepository productRepository;

    // Spring Cache abstraction — transparent caching
    @Cacheable(value = "products", key = "#productId")
    public Product getProduct(String productId) {
        log.info("Cache MISS: fetching from DB for productId={}", productId);
        return productRepository.findById(productId)
            .orElseThrow(() -> new ResourceNotFoundException("Product", productId));
    }

    @CacheEvict(value = "products", key = "#productId")
    public void invalidateProduct(String productId) {
        log.info("Cache EVICT: productId={}", productId);
    }

    @CachePut(value = "products", key = "#product.id") // update cache on write
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }
}
```

```java
// rate/RateLimiter.java
@Component
public class RateLimiter {
    private final RedisTemplate<String, Long> redisTemplate;

    /**
     * Fixed window rate limiting using Redis INCR.
     * INCR is atomic → no race condition.
     */
    public boolean isAllowed(String key, int limit, Duration window) {
        Long count = redisTemplate.opsForValue().increment(key);
        if (count == 1) {
            // Set TTL only on first request (to start the window)
            redisTemplate.expire(key, window);
        }
        return count <= limit;
    }
}
```

```java
// lock/RedisDistributedLock.java
@Component
public class RedisDistributedLock {
    private final StringRedisTemplate redis;

    public String tryAcquire(String lockKey, Duration ttl) {
        String lockValue = UUID.randomUUID().toString(); // unique owner identifier
        Boolean acquired = redis.opsForValue().setIfAbsent(lockKey, lockValue, ttl);
        return Boolean.TRUE.equals(acquired) ? lockValue : null;
    }

    /**
     * Atomic release: check owner AND delete in single operation.
     * Without Lua: check → delete has TOCTOU race condition
     *   Thread A: check(key == myValue) → true
     *   Thread B: lock expires, Thread C acquires it
     *   Thread A: delete(key) → deletes Thread C's lock!
     */
    public boolean release(String lockKey, String lockValue) {
        String luaScript = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
            """;
        Long result = redis.execute(
            new DefaultRedisScript<>(luaScript, Long.class),
            List.of(lockKey),
            lockValue
        );
        return Long.valueOf(1L).equals(result);
    }

    /** Try-with-lock helper */
    public boolean withLock(String key, Duration ttl, Runnable action) {
        String lockValue = tryAcquire(key, ttl);
        if (lockValue == null) return false;
        try {
            action.run();
            return true;
        } finally {
            release(key, lockValue);
        }
    }
}
```

---

## 4. Lab Steps

1. **Cache miss/hit:** `GET /products/123` lần 1 → log "Cache MISS". Lần 2 → không có log (cache hit). `redis-cli GET "products::123"` để xem cached value.
2. **Rate limiting:** Gọi endpoint 101 lần/phút → lần 101 nhận 429. `redis-cli GET "rate:ip:127.0.0.1"` → 101.
3. **Kafka consumer idempotency:** Publish cùng message 2 lần (same orderId) → consumer log "Duplicate ... skipping" lần 2.
4. **DLQ test:** Force exception trong consumer → sau 3 retries → check `orders.created.DLT` topic trong Kafka UI.
5. **Distributed lock:** 2 threads cùng try acquire same key → chỉ 1 thành công.
6. **Consumer lag:** Producer nhanh hơn consumer → monitor lag trong Kafka UI.

---

## 5. Checklist tự kiểm tra

- [ ] Redis `INCR` atomic — rate limiter không có race condition
- [ ] Distributed lock release: Lua script (atomic check-and-delete) — không phải GET rồi DEL riêng
- [ ] Kafka key = orderId → same order always same partition → ordering guarantee
- [ ] Consumer idempotency check TRƯỚC processing (not after)
- [ ] DLQ: DefaultErrorHandler với `FixedBackOff(1000L, 3)` → 3 retries → `.DLT` topic
- [ ] `@CacheEvict` sau write — không để stale data trong cache
- [ ] Consumer group rebalance khi thêm/bớt consumers → tạm thời stop consuming
