# Phase 15 — Production & Performance

> **Mục tiêu:** Tổng hợp tất cả 14 phase trước vào một production-grade system. Biết deploy, monitor, reproduce incidents, và make trade-off decisions.

---

## Part 1 — Observability (The Three Pillars)

### Logs — Structured Logging
```java
// logback-spring.xml với logstash encoder
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
    <customFields>{"service":"order-service","env":"prod"}</customFields>
</encoder>

// Code
log.info("Order placed",
    kv("orderId", order.getId()),
    kv("userId", order.getUserId()),
    kv("amount", order.getTotal()),
    kv("traceId", tracer.currentSpan().context().traceId()));
```

**Log fields quan trọng:**
- `traceId` / `spanId` — correlation
- `service`, `env`, `version` — context
- `userId`, `requestId` — business context
- `duration`, `status` — performance

### Metrics — Micrometer + Prometheus
```java
// Custom metrics
@Component
class OrderMetrics {
    private final Counter ordersCreated;
    private final Timer orderProcessingTime;
    private final Gauge activeOrders;

    OrderMetrics(MeterRegistry registry) {
        ordersCreated = Counter.builder("orders.created")
            .tag("status", "success")
            .register(registry);
        orderProcessingTime = Timer.builder("order.processing.time")
            .register(registry);
    }

    void recordOrderCreated() { ordersCreated.increment(); }
    void recordProcessingTime(Duration d) { orderProcessingTime.record(d); }
}
```

```yaml
# Prometheus scrape config
scrape_configs:
  - job_name: 'spring-boot'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['app:8080']
```

**Key metrics để monitor:**
| Metric | Alert khi |
|--------|---------|
| `jvm_memory_used_bytes` | > 85% heap |
| `hikaricp_connections_active` | Near pool max |
| `http_server_requests_seconds_max` | P99 > SLA |
| `kafka_consumer_records_lag` | Increasing trend |
| `resilience4j_circuitbreaker_state` | OPEN |

### Traces — OpenTelemetry
```java
// Auto-instrumented với spring-boot-starter-actuator + micrometer-tracing-bridge-otel
// Manual span khi cần
@Autowired Tracer tracer;

Span span = tracer.nextSpan().name("process-payment");
try (Tracer.SpanInScope scope = tracer.withSpan(span.start())) {
    span.tag("orderId", orderId.toString());
    processPayment(orderId);
} catch (Exception e) {
    span.error(e);
    throw e;
} finally {
    span.end();
}
```

---

## Part 2 — Deployment

### Docker
```dockerfile
# Multi-stage build
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY . .
RUN ./mvnw package -DskipTests

FROM eclipse-temurin:21-jre AS runtime
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar

# Security: non-root user
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
USER appuser

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-Djava.security.egd=file:/dev/./urandom", \
    "-jar", "app.jar"]
```

### Docker Compose (local stack)
```yaml
services:
  app:
    build: .
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/appdb
      SPRING_REDIS_HOST: redis
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_started }
    ports: ["8080:8080"]

  postgres:
    image: postgres:16
    environment: { POSTGRES_DB: appdb, POSTGRES_PASSWORD: secret }
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]

  redis:
    image: redis:7-alpine

  prometheus:
    image: prom/prometheus
    volumes: ["./prometheus.yml:/etc/prometheus/prometheus.yml"]

  grafana:
    image: grafana/grafana
    ports: ["3000:3000"]
```

### Graceful Shutdown
```yaml
# application.yml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

```java
// Spring handles SIGTERM:
// 1. Stop accepting new requests
// 2. Drain in-flight requests (up to 30s)
// 3. Shutdown Kafka consumers
// 4. Close DB connections
// 5. JVM exits
```

---

## Part 3 — Kubernetes Basics

```yaml
# deployment.yml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
  template:
    spec:
      containers:
        - name: order-service
          image: nguyenngoc/order-service:1.0.0
          resources:
            requests: { memory: "512Mi", cpu: "500m" }
            limits:   { memory: "1Gi",  cpu: "1000m" }
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
            initialDelaySeconds: 30
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            periodSeconds: 30
          env:
            - name: SPRING_DATASOURCE_URL
              valueFrom:
                secretKeyRef: { name: db-secret, key: url }
```

---

## Part 4 — Incident Playbook (Reproduce & Debug)

### Memory Leak
```
Triệu chứng: heap tăng dần, GC runs thường xuyên, cuối cùng OOM
Reproduce: static cache grow without eviction
Debug:
  1. jstat -gc <pid> 5000  → xem Old Gen tăng
  2. jcmd <pid> GC.heap_info
  3. jmap -dump:format=b,file=heap.hprof <pid>
  4. Eclipse MAT → Dominator Tree → tìm root cause
Fix: TTL cache, WeakReference, bounded collections
```

### Thread Pool Exhaustion
```
Triệu chứng: requests timeout, thread pool full errors
Reproduce: blocking calls in thread pool without timeout
Debug:
  1. jstack <pid>  → nhiều threads BLOCKED/WAITING
  2. /actuator/metrics/executor.active → near max
Fix: set timeout, increase pool (careful), async with CompletableFuture
```

### Connection Pool Exhaustion (HikariCP)
```
Triệu chứng: "Connection is not available, request timed out"
Reproduce: long transactions, N+1 queries, no connection timeout
Debug:
  1. HikariCP metrics: hikaricp_connections_active, pending
  2. pg_stat_activity → xem connections đang làm gì
Fix: shorter transactions, fix N+1, tune pool size, set connection timeout
```

### Cache Stampede
```
Triệu chứng: DB spike sau khi cache expires, response time spike
Reproduce: expire popular cache key, hammer with requests
Fix: distributed lock on cache refresh, probabilistic early expiration
```

### Cascading Failure
```
Triệu chứng: một service down → toàn bộ hệ thống chậm/down
Reproduce: shutdown một dependency, không có circuit breaker
Debug: trace requests → tìm service đang block
Fix: circuit breaker, timeout, bulkhead, fallback
```

### N+1 in Production
```
Triệu chứng: API chậm bất thường, DB query count cao
Reproduce: load test với paginated list endpoint
Debug:
  1. APM (Datadog/NewRelic) → xem query count per request
  2. show-sql=true local
Fix: FETCH JOIN, EntityGraph, BatchSize
```

---

## Part 5 — Trade-off Decisions

**REST vs Kafka:**
```
REST:
  + Simple, synchronous, easy to debug
  - Coupling, cascading failure risk
  Use: queries, real-time responses needed

Kafka:
  + Decoupling, scalability, replay
  - Eventual consistency, complexity
  Use: commands, workflows, audit events
```

**Session vs JWT:**
```
Session:
  + Instant revoke, server controls
  - State, not horizontally scalable without shared store
  Use: Server-rendered apps, need instant logout

JWT:
  + Stateless, microservices friendly
  - Cannot revoke before expiry (without blacklist)
  Use: REST APIs, microservices (với refresh token rotation)
```

**Redis vs Database:**
```
Redis:
  + Sub-millisecond, in-memory, data structures
  - Volatile (configure persistence), memory limited
  Use: cache, sessions, rate limiting, distributed lock

Database:
  + Persistent, ACID, relations
  - Slower, disk-bound
  Use: source of truth, complex queries
```

**Monolith vs Microservices:**
```
Monolith:
  + Simple deployment, no network latency, ACID transactions
  - Scale all-or-nothing, team coordination at scale
  Use: Early stage, small team, unclear domain

Microservices:
  + Independent scale, independent deploy
  - Network complexity, distributed transactions, ops overhead
  Use: Large teams, clear domain boundaries, different scaling needs
```

---

## Project — Production-Grade System

**Lấy bất kỳ project từ phase trước và đưa lên production standard.**

```
src/
├── (application code from previous phases)
├── observability/
│   ├── MetricsConfig.java
│   ├── TracingConfig.java
│   └── RequestMetricsFilter.java
├── health/
│   └── CustomHealthIndicator.java
└── resources/
    ├── logback-spring.xml          ← structured JSON logging
    └── application.yml             ← actuator, metrics config

ops/
├── Dockerfile
├── docker-compose.yml              ← full local stack
├── prometheus.yml
├── grafana/
│   └── dashboards/
│       └── app-dashboard.json
└── k8s/
    ├── deployment.yml
    ├── service.yml
    ├── configmap.yml
    └── secret.yml
```

**Definition of Done cho phase 15:**
- [ ] Structured JSON logs với correlation ID
- [ ] Custom metrics (business + technical)
- [ ] Grafana dashboard: latency, throughput, error rate
- [ ] Distributed tracing với Zipkin/Jaeger
- [ ] Docker image, multi-stage build
- [ ] Graceful shutdown works
- [ ] K8s health probes configured
- [ ] Reproduce + fix 3 incidents từ playbook trên
- [ ] Document 3 trade-off decisions với reasoning

---

## Checklist — 6 câu hỏi

Ví dụ với `Graceful Shutdown`:

| # | Câu hỏi | Trả lời |
|---|---------|---------|
| 1 | Là gì? | Process xử lý SIGTERM bằng cách drain in-flight work trước khi exit |
| 2 | Giải quyết gì? | Tránh dropped requests và data corruption khi deploy/restart |
| 3 | Hoạt động thế nào? | Spring stops accepting new requests → waits for active requests → closes resources |
| 4 | Khi nào cần? | Rolling deployment, autoscaling, pod eviction trong K8s |
| 5 | Khi nào không đủ? | Long-running jobs cần checkpointing; Kafka consumers cần commit offsets |
| 6 | Debug thế nào? | Kiểm tra `server.shutdown=graceful`; test bằng kill -TERM; monitor in-flight requests drop to 0 |

---

## Run

```bash
# Build Docker image
mvn spring-boot:build-image -pl phase-15-production-performance

# Run full stack
docker compose up -d

# Run K8s locally (minikube)
kubectl apply -f ops/k8s/

# Load test
k6 run ops/loadtest/script.js
```
