# Phase 15 — Production & Performance · Lý thuyết & Lab

---

## 1. Lý thuyết

### 1.1 Observability — Ba trụ cột

**Logs, Metrics, Traces — tại sao cần cả ba:**

```
Incident: API P99 latency spike at 14:30

Metrics: "P99 latency tăng từ 100ms lên 2000ms lúc 14:30"
    → Biết CÓ vấn đề, WHEN nó xảy ra

Logs: "14:30 OrderService ERROR: Connection pool exhausted"  
    → Biết VẤN ĐỀ là gì, ở đâu

Traces: Request abc123 → OrderService 1950ms → DBQuery 1900ms (slow!)
    → Biết ROOT CAUSE: DB query chậm
```

**Structured Logging — tại sao JSON thay vì plain text:**
```
# Plain text — khó parse, khó filter
2024-01-15 14:30:01 [main] ERROR UserService - Failed to create user alice@test.com

# Structured JSON — machine-parseable, searchable trong Elasticsearch/Loki
{"@timestamp":"2024-01-15T14:30:01Z","level":"ERROR","logger":"UserService",
 "message":"Failed to create user","email":"alice@test.com","traceId":"abc123","requestId":"xyz"}
```

**Distributed tracing tại sao cần:**
Một request HTTP đi qua 5 services. Chỉ service 4 log error. Làm sao biết service 1, 2, 3 đã làm gì với request đó? Tracing dùng `traceId` xâu chuỗi tất cả logs và spans lại.

---

### 1.2 Metrics — Lý thuyết Measurement

**Four Golden Signals (Google SRE):**
1. **Latency:** Thời gian xử lý request. Phân biệt success latency và error latency.
2. **Traffic:** Số requests/second vào system.
3. **Errors:** Rate của failed requests (4xx, 5xx).
4. **Saturation:** Mức độ "đầy" của resources (CPU %, DB connection pool %, queue depth).

**Percentiles vs Average:**
Average latency bị skew bởi outliers:
```
Request times: 10ms, 10ms, 10ms, 10ms, 5000ms (timeout)
Average: 1008ms ← misleading!
P50: 10ms ← most users experience
P99: 5000ms ← worst 1% experience
```

**P99 latency là SLA metric quan trọng nhất.** Nếu P99 = 2s: 1% users chờ 2 giây — với 10K req/s → 100 users/s đang chờ 2 giây.

**Prometheus metric types:**
- **Counter:** Tăng đều, không giảm. `http_requests_total`. Dùng `rate()` để tính rate.
- **Gauge:** Giá trị hiện tại, có thể tăng/giảm. `jvm_memory_used_bytes`, `db_connections_active`.
- **Histogram:** Phân phối giá trị, supports percentiles. `http_request_duration_seconds`.
- **Summary:** Như Histogram nhưng tính percentile tại client (không query được cross-instances).

---

### 1.3 Docker Multi-Stage Build — Tại sao

**Single-stage build vấn đề:**
```dockerfile
FROM eclipse-temurin:21-jdk
# JDK image: ~600MB
COPY . .
RUN ./mvnw package
# Final image: 600MB base + Maven + source code + test deps = 1.2GB
```

**Multi-stage build:**
```dockerfile
# Stage 1: Build (large, có Maven, source code, test jars)
FROM eclipse-temurin:21-jdk AS build
# ... compile và package

# Stage 2: Runtime (chỉ JRE + compiled jar)
FROM eclipse-temurin:21-jre-alpine AS runtime
COPY --from=build /app/target/app.jar app.jar
# JRE image: ~100MB. Final: ~120MB
```

**Benefits:**
- Smaller image → faster pull, less storage, smaller attack surface
- No source code in production image
- Separate build and runtime dependencies

**Layer caching — tại sao copy pom.xml trước:**
```dockerfile
# Copy pom.xml (changes rarely) → download deps → cache this layer
COPY pom.xml .
RUN ./mvnw dependency:go-offline

# Copy source code (changes frequently) → rebuild from this layer
COPY src ./src
RUN ./mvnw package -DskipTests
```
Nếu chỉ code thay đổi (không phải deps) → `mvnw dependency:go-offline` layer được cache → build nhanh.

---

### 1.4 Kubernetes — Core Concepts

**Pod:** Smallest deployable unit. 1+ containers sharing network namespace và volumes. Thường 1 pod = 1 container.

**Deployment:** Manages Pod replicas. Ensures N pods luôn running. Handles rolling updates và rollbacks.

**Service:** Load balancer trước pods. Fixed IP/DNS mặc dù pods đến rồi đi.

**Readiness vs Liveness Probe:**
```
Readiness Probe: "Pod sẵn sàng nhận traffic không?"
  - FAIL → K8s không route traffic đến pod này (nhưng không restart)
  - Dùng cho: slow startup, warming up caches, downstream dependencies

Liveness Probe: "Pod còn sống không?"
  - FAIL → K8s restart container
  - Dùng cho: detect deadlocks, memory leaks causing unresponsiveness
```

**Tại sao readiness và liveness probe KHÁC nhau:**
```
Scenario: Service đang khởi động, chưa ready
- Readiness FAIL → K8s không route traffic → good (không send request to unready pod)
- Liveness CHECK → should PASS → don't restart (let it finish starting up)

Nếu dùng liveness cho cả hai:
- App starts slowly → liveness fails → K8s restarts → infinite restart loop!
```

**Rolling update strategy:**
```yaml
strategy:
  rollingUpdate:
    maxSurge: 1        # Start 1 new pod before removing old
    maxUnavailable: 0  # Never reduce available pod count
# Result: N+1 pods temporarily → new pod ready → remove old → repeat
# Zero downtime deployment
```

---

### 1.5 Graceful Shutdown — Tại sao quan trọng

**Vấn đề không có graceful shutdown:**
```
Load Balancer: routing requests to Pod A
K8s: sends SIGTERM to Pod A (deployment update)
Pod A: kills immediately
→ In-flight requests (đang xử lý) → connection reset → client error!
→ DB transactions → rollback
→ Kafka messages being processed → unacked → reprocessing
```

**With graceful shutdown:**
```
K8s: sends SIGTERM
Pod A: 
  1. Stops accepting new connections (Readiness probe fails → LB stops routing)
  2. Waits for in-flight requests to complete (up to 30s)
  3. Closes DB connections properly
  4. Commits Kafka offsets
  5. Exits cleanly
K8s: after 30s (terminationGracePeriodSeconds), sends SIGKILL if still running
```

---

## 2. Vấn đề thường gặp & Cách fix (Incident Playbook)

### Incident 1: Memory Leak — OOM Crash

**Symptoms:** 
- Heap usage tăng steadily, không giảm sau GC
- Full GC frequency tăng
- Eventually: `java.lang.OutOfMemoryError: Java heap space`

**Diagnosis:**
```bash
# 1. Enable automatic heap dump on OOM (set at startup)
# -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heap.hprof

# 2. Monitor live với jstat
jstat -gc $(jps | grep YourApp | awk '{print $1}') 5000
# Watch OU (Old Used) column — nếu tăng steadily → leak

# 3. Force GC để xem nếu data actually release
jcmd <pid> GC.run
# Nếu OU vẫn cao sau GC → objects held by GC roots = leak

# 4. Heap histogram — top memory consumers
jcmd <pid> GC.heap_info
jmap -histo:live <pid> | head -30

# 5. Full heap dump analysis (dùng Eclipse MAT)
jmap -dump:format=b,file=/tmp/heap.hprof <pid>
# Mở heap.hprof → Leak Suspects Report
# Dominator Tree → objects holding most memory
```

**Common root causes & fixes:**
```java
// Root cause 1: Static Map without eviction
// ❌
private static final Map<String, Session> sessions = new HashMap<>();
// ✅ LRU cache với max size
private static final Map<String, Session> sessions = Collections.synchronizedMap(
    new LinkedHashMap<>(1000, 0.75f, true) {
        protected boolean removeEldestEntry(Map.Entry e) { return size() > 1000; }
    }
);

// Root cause 2: ThreadLocal not cleared
// ❌
ThreadLocal<HeavyObject> local = new ThreadLocal<>();
void serve(Request req) { local.set(new HeavyObject()); process(); }
// ✅
void serve(Request req) {
    try { local.set(new HeavyObject()); process(); }
    finally { local.remove(); } // MUST remove
}

// Root cause 3: Unclosed streams/connections
// ✅ Always try-with-resources
try (Connection conn = dataSource.getConnection()) { ... }
```

---

### Incident 2: Thread Pool Exhaustion — Requests Timeout

**Symptoms:**
- Requests queue up, eventually timeout
- `executor.active` metric = `maximumPoolSize`
- Thread dump shows many threads `WAITING` or `BLOCKED`

**Diagnosis:**
```bash
# 1. Actuator metrics
curl http://localhost:8080/actuator/metrics/executor.active
curl http://localhost:8080/actuator/metrics/executor.queue.remaining

# 2. Thread dump — identify what threads are doing
jcmd <pid> Thread.print > threads.txt

# Patterns to look for:
# Many threads in same state → system-wide issue
grep -A 5 "BLOCKED" threads.txt | head -50
grep -A 5 "WAITING" threads.txt | head -50

# If many threads waiting on DB → connection pool exhausted, or slow query
# If many BLOCKED on same lock → contention issue
```

**Root causes & fixes:**
```java
// Root cause 1: Slow downstream service (DB, external API) blocking threads
// ❌ No timeout → thread blocked indefinitely
productService.getProduct(id); // might hang

// ✅ Add timeout
try {
    return CompletableFuture.supplyAsync(() -> productService.getProduct(id))
        .get(5, TimeUnit.SECONDS);
} catch (TimeoutException e) {
    return ProductResponse.fallback(id);
}

// Root cause 2: N+1 query causing slow DB transactions
// Each thread waits longer → fewer can run simultaneously
// Fix: Add FETCH JOIN (see Phase 11)

// Root cause 3: Thread pool too small for load
// Fix: Tune pool size — but NOT just increase! Understand root cause first
```

---

### Incident 3: High Latency — P99 Spike

**Systematic diagnosis:**
```bash
# 1. What changed? Recent deployment? Traffic spike?
git log --oneline -10
kubectl rollout history deployment/my-service

# 2. Which endpoint is slow?
# Grafana: filter http_server_requests_seconds by uri

# 3. Is it DB?
# PG slow query log:
ALTER SYSTEM SET log_min_duration_statement = 1000; -- log queries > 1s
SELECT pg_reload_conf();
# Check /var/log/postgresql/postgresql.log

# 4. Is it GC?
# Check GC log for Full GC pauses
grep "Full GC" /tmp/gc.log | tail -20

# 5. Is it external service? (Kafka, Redis, external API)
# Trace request in Zipkin — which span is long?

# 6. Is it cache miss rate spike? (after deployment: cache cold start)
# Redis: redis-cli --stat | grep miss
```

---

## 3. Code mẫu

```xml
<!-- pom.xml: Logstash encoder for structured logging -->
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

```xml
<!-- src/main/resources/logback-spring.xml -->
<configuration>
    <springProfile name="prod">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
                <!-- Service metadata in every log line -->
                <customFields>{"service":"order-service","env":"prod"}</customFields>
                <fieldNames>
                    <timestamp>@timestamp</timestamp>
                    <version>[ignore]</version>
                </fieldNames>
            </encoder>
        </appender>
    </springProfile>

    <springProfile name="!prod">
        <!-- Human-readable in dev -->
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} [%X{traceId}] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
    </springProfile>

    <root level="INFO"><appender-ref ref="CONSOLE"/></root>
</configuration>
```

```java
// observability/OrderMetrics.java — Custom business metrics
@Component
public class OrderMetrics {

    private final Counter ordersCreated;
    private final Counter ordersFailed;
    private final Timer orderProcessingTime;
    private final AtomicInteger pendingOrders = new AtomicInteger(0);

    public OrderMetrics(MeterRegistry registry) {
        // Counter: increment-only, for rate queries
        ordersCreated = Counter.builder("orders.created.total")
            .description("Total number of orders successfully created")
            .tag("service", "order-service")
            .register(registry);

        ordersFailed = Counter.builder("orders.failed.total")
            .description("Total number of failed order attempts")
            .register(registry);

        // Timer: records duration + count. Supports percentiles.
        orderProcessingTime = Timer.builder("orders.processing.duration")
            .description("Time to process an order")
            .publishPercentiles(0.5, 0.95, 0.99) // P50, P95, P99
            .register(registry);

        // Gauge: reflects current value (not cumulative)
        Gauge.builder("orders.pending.count", pendingOrders, AtomicInteger::get)
            .description("Number of orders currently being processed")
            .register(registry);
    }

    public void recordOrderCreated() { ordersCreated.increment(); }
    public void recordOrderFailed() { ordersFailed.increment(); }
    public void recordProcessingTime(Duration d) { orderProcessingTime.record(d); }
    public void incrementPending() { pendingOrders.incrementAndGet(); }
    public void decrementPending() { pendingOrders.decrementAndGet(); }
}
```

```dockerfile
# ops/Dockerfile — Multi-stage
# Stage 1: Build (includes JDK, Maven, all deps, source code)
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app

# Copy build files first (cached if unchanged → fast rebuild)
COPY pom.xml mvnw ./
COPY .mvn .mvn
RUN ./mvnw dependency:go-offline -q  # ← this layer cached if pom.xml unchanged

# Now copy source (invalidates cache here, but deps already downloaded)
COPY src ./src
RUN ./mvnw package -DskipTests -q

# Stage 2: Runtime (only JRE — no JDK, no Maven, no source, no test deps)
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app

# Security: run as non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=build /app/target/*.jar app.jar

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=30s --retries=3 \
    CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java",
    # Container-aware memory settings (NEVER hardcode -Xmx in container)
    "-XX:+UseContainerSupport",
    "-XX:MaxRAMPercentage=75.0",
    # G1GC with 200ms pause target
    "-XX:+UseG1GC",
    "-XX:MaxGCPauseMillis=200",
    # On OOM: dump heap and exit (so K8s restarts)
    "-XX:+HeapDumpOnOutOfMemoryError",
    "-XX:HeapDumpPath=/tmp/",
    # GC logging (to file, doesn't affect stdout)
    "-Xlog:gc*:file=/tmp/gc.log:tags,time:filesize=100m,filecount=3",
    "-jar", "app.jar"]
```

```yaml
# ops/k8s/deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # zero-downtime
  template:
    spec:
      containers:
        - name: order-service
          image: myregistry/order-service:1.0.0
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"  # OOM killed if exceeded → pod restart

          # Different endpoints for different purposes!
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20  # give app time to start
            periodSeconds: 5
            failureThreshold: 3  # 3 consecutive failures → stop routing

          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60  # longer than readiness — let app fully warm up
            periodSeconds: 30

          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
            # Secrets from K8s Secrets (not plain env vars!)
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
```

```yaml
# application.yml — Graceful shutdown
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # wait max 30s for in-flight requests

management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus,info
  health:
    readinessState:
      enabled: true
    livenessState:
      enabled: true
  metrics:
    export:
      prometheus:
        enabled: true
  tracing:
    sampling:
      probability: 0.1  # 10% in prod (100% too expensive at scale)
```

---

## 4. Grafana — Key Queries

```promql
# Request rate (req/s per endpoint)
rate(http_server_requests_seconds_count{application="order-service"}[1m])

# Error rate (%)
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
/ sum(rate(http_server_requests_seconds_count[5m])) * 100

# P99 latency
histogram_quantile(0.99, 
  rate(http_server_requests_seconds_bucket[5m])
)

# JVM heap usage (%)
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} * 100

# DB connection pool saturation
hikaricp_connections_active / hikaricp_connections_max * 100

# Kafka consumer lag
kafka_consumer_records_lag{group="order-processing"}
```

---

## 5. Lab Steps

1. **Structured logs:** Start app with `--spring.profiles.active=prod` → request → see JSON logs with traceId
2. **Prometheus:** `GET /actuator/prometheus` → find `orders_created_total` metric
3. **Grafana:** Setup dashboard with 4 panels: request rate, error rate, P99 latency, JVM heap
4. **Multi-stage build:** `docker build` → check image size vs single-stage
5. **K8s readiness:** Deploy with wrong DB password → readiness probe fails → no traffic routes
6. **Graceful shutdown:** Start load test → `kubectl rollout restart deployment` → observe 0 errors
7. **Incident simulation:** Create memory leak (static Map + add items) → monitor heap in Grafana → trigger heap dump

---

## 6. Checklist — Production Readiness

**Observability:**
- [ ] Structured JSON logs với: traceId, requestId, userId, service, env
- [ ] MDC cleared in finally block in every filter
- [ ] Business metrics: `orders.created.total`, `orders.failed.total`, `orders.pending.count`
- [ ] P50/P95/P99 latency metrics
- [ ] Grafana dashboard: 4 golden signals (latency, traffic, errors, saturation)
- [ ] Distributed trace in Zipkin for cross-service requests

**Deployment:**
- [ ] Multi-stage Dockerfile — image < 200MB
- [ ] Non-root user in container
- [ ] `HEALTHCHECK` in Dockerfile
- [ ] `-XX:+UseContainerSupport` + `-XX:MaxRAMPercentage=75.0`
- [ ] GC logging enabled to file
- [ ] `-XX:+HeapDumpOnOutOfMemoryError` enabled

**Kubernetes:**
- [ ] Readiness probe → `/actuator/health/readiness` (exclude downstream)
- [ ] Liveness probe → `/actuator/health/liveness` (minimal check)
- [ ] Resource requests AND limits set
- [ ] Secrets from K8s Secrets, not plain env vars
- [ ] Rolling update: `maxUnavailable=0`
- [ ] `terminationGracePeriodSeconds` > app shutdown timeout

**Resilience:**
- [ ] Graceful shutdown: 30s timeout for in-flight requests
- [ ] Can reproduce Memory Leak incident: detect + fix
- [ ] Can reproduce Thread Pool Exhaustion: detect + fix
- [ ] Circuit breaker configured for external calls
