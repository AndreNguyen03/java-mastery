# Java Mastery

Multi-module Maven monorepo — Java + Spring Boot Backend roadmap, 15 phases.

---

## Nguyên tắc xuyên suốt

```
CONCEPT → WHY DOES IT EXIST? → HOW DOES IT WORK?
    → USE IT IN CODE → TEST IT → BREAK / REPRODUCE
    → DEBUG / PROFILE → OPTIMIZE → UNDERSTAND TRADE-OFF
```

Một concept chỉ được coi là nắm khi trả lời được 6 câu:
1. Nó là gì?
2. Nó giải quyết vấn đề gì?
3. Nó hoạt động như thế nào?
4. Khi nào nên dùng?
5. Khi nào không nên dùng?
6. Nếu hệ thống có vấn đề, debug thế nào?

---

## Roadmap

| Phase | Module | Project | Stack |
|-------|--------|---------|-------|
| 01 | [Java Language](java-language/README.md) | CLI Expense Tracker | Java 21 |
| 02 | [DSA & Collections](dsa-collections/README.md) | Algorithm Lab | Java 21 |
| 03 | [Functional Java](functional-java/README.md) | Data Processing Engine | Java 21 |
| 04 | [OOP & Design Patterns](oop-design-patterns/README.md) | Banking Engine | Java 21 |
| 05 | [Exception, Logging & Testing](exception-logging-testing/README.md) | Test phase 01/04 | JUnit 5, Mockito |
| 06 | [JVM, Memory & Performance](jvm-memory-performance/README.md) | JVM Performance Lab | JFR, VisualVM |
| 07 | [Concurrency](concurrency/README.md) | Concurrent Task Scheduler | java.util.concurrent |
| 08 | [I/O & Networking](io-networking/README.md) | Mini HTTP Client/Server | NIO, java.net.http |
| 09 | [Spring Core](spring-core/README.md) | Spring Experiments | Spring 6 (no Boot) |
| 10 | [Spring Boot](spring-boot/README.md) | Project Management System | Spring Boot 3 |
| 11 | [Database, JPA & Hibernate](database-jpa-hibernate/README.md) | PMS + PostgreSQL | JPA, Liquibase, Testcontainers |
| 12 | [Spring Security](spring-security/README.md) | Identity & Auth System | JWT, RBAC |
| 13 | [Redis, Kafka & Messaging](redis-kafka-messaging/README.md) | Order Processing System | Redis, Kafka |
| 14 | [Microservices & Distributed](microservices-distributed-system/README.md) | E-commerce Microservices | Spring Cloud, Saga |
| 15 | [Production & Performance](production-performance/README.md) | Production-grade System | Docker, K8s, Prometheus |

---

## Requirements

- Java 21+
- Maven 3.9+
- Docker Desktop (từ phase 11 trở đi)

---

## Commands

```bash
# Build toàn bộ
mvn clean install

# Build 1 phase
mvn clean install -pl java-language

# Test 1 phase
mvn test -pl concurrency

# Run Spring Boot app
mvn spring-boot:run -pl spring-boot

# Skip tests (build nhanh)
mvn clean install -DskipTests
```

---

## Dependency Map

```
Phases 01–08  ← Java thuần (JUnit 5, Mockito, SLF4J/Logback)
Phase  09     ← Spring Framework 6 (no Boot)
Phases 10–15  ← Spring Boot 3 + ecosystem
```

```
01 → 02 → 03 → 04 ─┐
                    ↓
               05 (test everything above)
                    ↓
               06 (JVM — applies to all)
                    ↓
               07 → 08
                    ↓
               09 → 10 → 11 → 12 → 13 → 14 → 15
```
