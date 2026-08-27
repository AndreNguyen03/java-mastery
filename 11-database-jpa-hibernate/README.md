# Phase 11 — Database, JPA & Hibernate

> **Mục tiêu:** Viết code JPA hiệu quả và debug được vấn đề N+1, slow query, deadlock, transaction isolation — không chỉ biết `@Entity` và `findById()`.

---

## Part 1 — SQL nền tảng

### Queries quan trọng
```sql
-- JOIN types
SELECT u.name, p.title
FROM users u
INNER JOIN projects p ON p.owner_id = u.id;

-- GROUP BY + HAVING
SELECT u.id, COUNT(t.id) task_count
FROM users u
LEFT JOIN tasks t ON t.assignee_id = u.id
GROUP BY u.id
HAVING COUNT(t.id) > 5;

-- CTE (Common Table Expression)
WITH active_projects AS (
    SELECT * FROM projects WHERE status = 'ACTIVE'
)
SELECT ap.title, COUNT(t.id)
FROM active_projects ap
LEFT JOIN tasks t ON t.project_id = ap.id
GROUP BY ap.id;

-- Window function
SELECT name,
       salary,
       RANK() OVER (PARTITION BY department ORDER BY salary DESC) rank
FROM employees;
```

### Index
```sql
CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_tasks_project_status ON tasks(project_id, status);  -- composite

-- EXPLAIN ANALYZE để xem query plan
EXPLAIN ANALYZE
SELECT * FROM tasks WHERE project_id = 1 AND status = 'OPEN';
-- Seq Scan → no index; Index Scan → good
```

**Khi nào index giúp:** WHERE, JOIN ON, ORDER BY, GROUP BY columns.

**Khi nào không:** Columns có ít distinct values (e.g. boolean); bảng nhỏ.

---

## Part 2 — Transaction & Isolation

### ACID
| Property | Nghĩa |
|---------|-------|
| **Atomicity** | All or nothing |
| **Consistency** | DB rules (constraints, triggers) maintained |
| **Isolation** | Concurrent transactions không thấy intermediate state của nhau |
| **Durability** | Committed data persists |

### Isolation Levels
| Level | Dirty Read | Non-repeatable Read | Phantom Read |
|-------|-----------|---------------------|--------------|
| READ UNCOMMITTED | ✓ (có thể) | ✓ | ✓ |
| READ COMMITTED | ✗ | ✓ | ✓ |
| REPEATABLE READ | ✗ | ✗ | ✓ |
| SERIALIZABLE | ✗ | ✗ | ✗ |

PostgreSQL default: `READ COMMITTED`. Hầu hết apps dùng cái này.

**MVCC (Multi-Version Concurrency Control):** PostgreSQL giữ nhiều versions của row — readers không block writers.

### Locks
```sql
-- Optimistic: version field, check before update
UPDATE tasks SET title = ?, version = version + 1
WHERE id = ? AND version = ?
-- 0 rows affected → conflict

-- Pessimistic: lock row
SELECT * FROM tasks WHERE id = 1 FOR UPDATE;
-- Block other transactions
```

---

## Part 3 — JPA & Hibernate

### Entity Lifecycle
```
Transient   → new Task() (không có ID, JPA không track)
    ↓ persist()
Managed     → trong Persistence Context (JPA track changes)
    ↓ detach() / close session
Detached    → ngoài PC (changes không được sync)
    ↓ merge()
Managed     → lại (merge state vào PC)
    ↓ remove()
Removed     → được xoá khi flush
```

### Persistence Context & Dirty Checking
```java
@Transactional
void updateTaskTitle(Long id, String newTitle) {
    Task task = taskRepository.findById(id).orElseThrow();
    task.setTitle(newTitle);  // KHÔNG cần save() — dirty checking tự làm
    // Khi transaction commit: Hibernate so sánh snapshot vs current → tự UPDATE
}
```

**Flush:** Sync changes từ PC xuống DB. Xảy ra tự động trước query và commit.

---

## Part 4 — N+1 Problem

### Problem
```java
// Xấu — N+1
List<Project> projects = projectRepo.findAll();  // 1 query
for (Project p : projects) {
    p.getTasks().size();  // N queries (lazy load!)
}
```

### Solutions
```java
// Solution 1: FETCH JOIN (JPQL)
@Query("""
    SELECT DISTINCT p FROM Project p
    LEFT JOIN FETCH p.tasks
    WHERE p.status = :status
""")
List<Project> findWithTasks(@Param("status") ProjectStatus status);

// Solution 2: Entity Graph
@EntityGraph(attributePaths = {"tasks", "owner"})
List<Project> findAll();

// Solution 3: Batch size (Hibernate)
@BatchSize(size = 20)
@OneToMany private Set<Task> tasks;
// → SELECT * FROM tasks WHERE project_id IN (1,2,3,...20)
```

**Phát hiện N+1:** `spring.jpa.show-sql=true` + `logging.level.org.hibernate.SQL=DEBUG` → đếm số queries.

---

## Part 5 — @Transactional Deep Dive

```java
@Transactional(
    propagation  = Propagation.REQUIRED,      // join existing or create new (default)
    isolation    = Isolation.READ_COMMITTED,  // isolation level
    readOnly     = true,                       // optimization cho read-only
    rollbackFor  = {BusinessException.class}, // rollback cho checked exceptions
    timeout      = 30                          // seconds
)
```

### Propagation
| Value | Hành vi |
|-------|---------|
| `REQUIRED` | Join existing; create if none (default) |
| `REQUIRES_NEW` | Suspend existing; create new |
| `SUPPORTS` | Join if exists; no transaction if none |
| `NOT_SUPPORTED` | Suspend existing; no transaction |
| `MANDATORY` | Must join existing; exception if none |
| `NEVER` | Must not be in transaction; exception if yes |

### Self-invocation problem
```java
@Service
class OrderService {
    void createOrder(Order order) {
        this.confirmOrder(order.getId()); // KHÔNG qua proxy → @Transactional KHÔNG work
    }

    @Transactional
    void confirmOrder(Long id) { ... }
}
```

---

## Part 6 — Liquibase

```yaml
# application.yml
spring:
  liquibase:
    change-log: classpath:db/changelog/db.changelog-master.yaml
```

```yaml
# db.changelog-master.yaml
databaseChangeLog:
  - include:
      file: db/changelog/001-create-users.sql
  - include:
      file: db/changelog/002-create-projects.sql
```

```sql
-- 001-create-users.sql
--liquibase formatted sql
--changeset nguyenngoc:001
CREATE TABLE users (
    id         BIGSERIAL PRIMARY KEY,
    email      VARCHAR(255) NOT NULL UNIQUE,
    name       VARCHAR(100) NOT NULL,
    created_at TIMESTAMP   NOT NULL DEFAULT NOW()
);
--rollback DROP TABLE users;
```

---

## Project — Project Management System + PostgreSQL

**Nối tiếp Phase 10. Thêm real DB, JPA entities, migrations.**

```
src/
├── domain/
│   ├── user/
│   │   ├── User.java          ← @Entity
│   │   └── UserRepository.java ← JpaRepository
│   ├── project/
│   └── task/
├── resources/
│   └── db/changelog/
│       ├── db.changelog-master.yaml
│       ├── 001-create-users.sql
│       ├── 002-create-projects.sql
│       └── 003-create-tasks.sql
└── docker-compose.yml         ← PostgreSQL local
```

**Reproduce và fix các scenarios:**
```
Scenario 1: N+1                    → FETCH JOIN / EntityGraph
Scenario 2: Slow query             → EXPLAIN ANALYZE → add index
Scenario 3: Optimistic lock conflict → version field + retry
Scenario 4: Deadlock               → lock order consistency
Scenario 5: Connection pool exhausted → HikariCP config + timeout
Scenario 6: Transaction rollback không đúng → kiểm tra propagation
```

---

## Checklist — 6 câu hỏi

Ví dụ với `N+1`:

| # | Câu hỏi | Trả lời |
|---|---------|---------|
| 1 | Là gì? | 1 query lấy N entities, rồi N queries nữa để lấy associated data |
| 2 | Giải quyết gì? | N/A — đây là vấn đề cần tránh |
| 3 | Xảy ra thế nào? | LAZY fetch + access association ngoài transaction boundary |
| 4 | Phát hiện thế nào? | show-sql=true; Hibernate Statistics; Datadog query count |
| 5 | Fix thế nào? | FETCH JOIN, EntityGraph, hoặc BatchSize tùy use case |
| 6 | Khi nào EAGER lại xấu hơn? | EAGER fetch mọi query dù không cần → CartesianProduct khi nhiều collections |

---

## Run

```bash
docker compose up -d    # start PostgreSQL
mvn spring-boot:run -pl phase-11-database-jpa-hibernate
mvn test -pl phase-11-database-jpa-hibernate
```
