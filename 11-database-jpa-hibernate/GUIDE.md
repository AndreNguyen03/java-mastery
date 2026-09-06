# Phase 11 — Database, JPA & Hibernate · Lý thuyết & Lab

---

## 1. Lý thuyết

### 1.1 ACID & Transaction Isolation

**ACID properties:**
- **Atomicity:** Tất cả hoặc không có gì — nếu transfer tiền fail ở bước 2, bước 1 được rollback
- **Consistency:** DB luôn ở trạng thái hợp lệ theo constraints (foreign key, check constraints)
- **Isolation:** Concurrent transactions không "thấy" nhau's partial results
- **Durability:** Committed transaction không bị mất dù crash ngay sau commit

**Isolation levels và anomalies:**
| Level | Dirty Read | Non-repeatable Read | Phantom Read |
|---|---|---|---|
| READ UNCOMMITTED | ✅ possible | ✅ possible | ✅ possible |
| READ COMMITTED (PG default) | ❌ prevented | ✅ possible | ✅ possible |
| REPEATABLE READ | ❌ | ❌ prevented | ✅ possible |
| SERIALIZABLE | ❌ | ❌ | ❌ prevented |

- **Dirty Read:** Đọc uncommitted data của transaction khác → nếu kia rollback → đọc data chưa thực sự tồn tại
- **Non-repeatable Read:** Đọc cùng row 2 lần trong cùng transaction → giá trị khác nhau (vì transaction khác update)
- **Phantom Read:** Query cùng điều kiện 2 lần → số rows khác nhau (vì transaction khác INSERT)

**MVCC (Multi-Version Concurrency Control) — PostgreSQL:**
PostgreSQL không lock rows cho READ — thay vào đó lưu nhiều versions của row. Mỗi transaction thấy snapshot của DB tại thời điểm transaction bắt đầu. Writes vẫn cần lock. → Reads không block writes, writes không block reads.

---

### 1.2 JPA Entity Lifecycle

```
New (Transient) → [persist()] → Managed → [flush/commit] → Persistent (in DB)
                                   ↓                              ↓
                               [remove()]                   [merge()]
                                   ↓                              ↓
                               Removed ← ─────────────── Detached
                                                (outside of session)
```

**Managed state — Dirty Checking:**
Khi entity ở trong `EntityManager` context (Managed state), Hibernate theo dõi mọi thay đổi. Khi flush/commit: so sánh current state với snapshot tại lúc load → auto-generate UPDATE SQL cho changed fields. **Không cần gọi `save()` sau khi update managed entity!**

```java
@Transactional
public void updateStatus(Long id, Status newStatus) {
    Project project = repository.findById(id).orElseThrow();
    project.setStatus(newStatus); // entity is MANAGED — dirty check will handle this
    // No repository.save() needed!
} // flush on commit → UPDATE projects SET status=? WHERE id=?
```

**Detached state:** Khi entity ra khỏi session (transaction ended, serialized, etc.) — Hibernate không track changes nữa. Phải `merge()` để re-attach.

---

### 1.3 N+1 Problem — Diagnosis & Fix

**N+1 xảy ra khi:**
Load collection (1 query) → duyệt qua items → access lazy-loaded association (N queries).

```java
// 1 query: SELECT * FROM projects WHERE status='ACTIVE'
List<Project> projects = projectRepo.findByStatus(ACTIVE); // returns 50 projects

for (Project p : projects) {
    System.out.println(p.getOwner().getName()); // ← 50 queries! SELECT * FROM users WHERE id=?
    System.out.println(p.getTasks().size());    // ← 50 queries! SELECT * FROM tasks WHERE project_id=?
}
// Total: 1 + 50 + 50 = 101 queries. DB under heavy load.
```

**Cách detect N+1:**
1. `spring.jpa.show-sql=true` — đếm số queries trong log
2. `hibernate.generate_statistics=true` — stats per session
3. p6spy hoặc datasource-proxy — log mọi SQL với caller stack

**Fix options (theo ưu tiên):**

**Fix 1: FETCH JOIN** — tốt nhất cho specific query:
```java
@Query("SELECT DISTINCT p FROM Project p LEFT JOIN FETCH p.owner LEFT JOIN FETCH p.tasks WHERE p.status = :status")
List<Project> findByStatusWithAll(@Param("status") ProjectStatus status);
// 1 query: SELECT p.*, u.*, t.* FROM projects p LEFT JOIN users u ... LEFT JOIN tasks t ...
// DISTINCT vì JOIN với collection tạo duplicate rows
```

**Fix 2: EntityGraph** — declarative, reusable:
```java
@EntityGraph(attributePaths = {"owner", "tasks"})
List<Project> findByStatus(ProjectStatus status);
// Spring Data auto-generates FETCH JOIN
```

**Fix 3: @BatchSize** — batch loading (giảm N queries xuống N/batchSize):
```java
@OneToMany(mappedBy = "project")
@BatchSize(size = 20) // load 20 tasks collections at once
List<Task> tasks;
// 1 query SELECT * FROM projects, then 1-2 batch queries for tasks
```

**Khi nào không nên FETCH JOIN:**
Không FETCH JOIN nhiều collections trong cùng một query (cartesian product vấn đề). Ví dụ: project có 10 tasks và 5 members → join cả hai → 50 rows cho mỗi project.

---

### 1.4 `@Transactional` — Propagation

**Propagation types quan trọng:**
- `REQUIRED` (default): dùng existing transaction, tạo mới nếu không có
- `REQUIRES_NEW`: luôn tạo transaction mới, suspend existing → thường dùng cho audit log (phải commit ngay cả khi outer tx rollback)
- `SUPPORTS`: dùng existing nếu có, không tạo mới
- `NOT_SUPPORTED`: không dùng transaction (suspend existing)
- `NEVER`: lỗi nếu có existing transaction
- `MANDATORY`: lỗi nếu không có existing transaction

**`readOnly = true` optimization:**
```java
@Transactional(readOnly = true)
public List<Project> findAll() { ... }
```
Hibernate skip dirty checking và snapshot creation → save memory và CPU. DB có thể route đến read replica. KHÔNG thể UPDATE/INSERT trong readOnly transaction.

---

### 1.5 Optimistic vs Pessimistic Locking

**Optimistic Locking (`@Version`):**
Assume conflicts ít xảy ra. Check tại commit time. 
- DB column `version BIGINT`
- Hibernate: `UPDATE projects SET name=?, version=version+1 WHERE id=? AND version=?`
- Nếu version mismatch (0 rows updated) → `OptimisticLockingFailureException`
- Application phải retry hoặc inform user

**Pessimistic Locking:**
Lock row khi đọc. Ngăn concurrent writes.
```java
@Lock(LockModeType.PESSIMISTIC_WRITE) // SELECT ... FOR UPDATE
Optional<Project> findById(Long id);
// Blocks other transactions from reading this row
```
Tốt khi conflict rate cao. Nhược điểm: deadlock risk, reduced throughput.

---

## 2. Vấn đề thường gặp & Cách fix

### Issue 1: `LazyInitializationException`

```
org.hibernate.LazyInitializationException: could not initialize proxy — no Session
```

**Nguyên nhân:** Access lazy association NGOÀI transaction. Session đã closed.

```java
// ❌ Service returns entity, controller accesses lazy field
Project project = projectService.findById(id); // transaction ends here
return ProjectResponse.from(project); // project.getTasks() → LazyInitializationException!

// ✅ Fix 1: Fetch join in service WITHIN transaction
@Transactional
public ProjectResponse findById(Long id) {
    Project p = projectRepo.findByIdWithTasks(id).orElseThrow();
    return ProjectResponse.from(p); // map to DTO while still in transaction
}

// ✅ Fix 2: open-in-view=false (recommended) + always fetch what you need
```

### Issue 2: `@Transactional` self-invocation (same as Spring AOP)

### Issue 3: Liquibase vs Hibernate DDL

```yaml
# ❌ NEVER in production
spring.jpa.hibernate.ddl-auto: create-drop # drops DB on shutdown!
spring.jpa.hibernate.ddl-auto: create      # drops and recreates on startup

# ✅ Use these
spring.jpa.hibernate.ddl-auto: validate    # validates schema matches entities
spring.jpa.hibernate.ddl-auto: none        # don't touch schema (Liquibase manages it)
```

Liquibase mang lại: version controlled migrations, rollback scripts, audit trail.

---

## 3. Code mẫu

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: pms_db
      POSTGRES_USER: pms
      POSTGRES_PASSWORD: pms_secret
    ports: ["5432:5432"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U pms"]
      interval: 5s
```

```sql
-- 001-create-users.sql
--liquibase formatted sql
--changeset nguyenngoc:001
CREATE TABLE users (
    id         BIGSERIAL    PRIMARY KEY,
    email      VARCHAR(255) NOT NULL UNIQUE,
    name       VARCHAR(100) NOT NULL,
    created_at TIMESTAMP    NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_users_email ON users(email);
--rollback DROP TABLE users;

-- 002-create-projects.sql
--liquibase formatted sql
--changeset nguyenngoc:002
CREATE TABLE projects (
    id         BIGSERIAL    PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    status     VARCHAR(20)  NOT NULL DEFAULT 'DRAFT',
    deadline   DATE,
    owner_id   BIGINT       NOT NULL REFERENCES users(id),
    version    BIGINT       NOT NULL DEFAULT 0,
    created_at TIMESTAMP    NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_projects_owner ON projects(owner_id);
CREATE INDEX idx_projects_status ON projects(status);
--rollback DROP TABLE projects;
```

```java
// domain/Project.java
@Entity
@Table(name = "projects")
public class Project {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private ProjectStatus status = ProjectStatus.DRAFT;

    private LocalDate deadline;

    @ManyToOne(fetch = FetchType.LAZY)   // LAZY: don't load owner unless accessed
    @JoinColumn(name = "owner_id", nullable = false)
    private User owner;

    @Version  // Optimistic locking — Hibernate manages this column
    private Long version;

    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @OneToMany(mappedBy = "project", fetch = FetchType.LAZY, cascade = CascadeType.ALL)
    private List<Task> tasks = new ArrayList<>();

    protected Project() {} // JPA requirement

    public Project(String name, User owner, LocalDate deadline) {
        this.name = name;
        this.owner = owner;
        this.deadline = deadline;
    }

    @PrePersist protected void onCreate() { createdAt = LocalDateTime.now(); }

    // Getters, updateStatus...
}
```

```java
// domain/ProjectRepository.java
public interface ProjectRepository extends JpaRepository<Project, Long> {

    // ❌ This will N+1 when accessing project.getOwner() or project.getTasks()
    List<Project> findByStatus(ProjectStatus status);

    // ✅ Fix: FETCH JOIN
    @Query("""
        SELECT DISTINCT p FROM Project p
        LEFT JOIN FETCH p.owner
        LEFT JOIN FETCH p.tasks
        WHERE p.status = :status
    """)
    List<Project> findByStatusWithAll(@Param("status") ProjectStatus status);

    // ✅ Alternative: EntityGraph (annotation-based)
    @EntityGraph(attributePaths = {"owner", "tasks"})
    List<Project> findAll();
}
```

```java
// domain/ProjectService.java
@Service
public class ProjectService {

    private final ProjectRepository projectRepository;

    public ProjectService(ProjectRepository projectRepository) {
        this.projectRepository = projectRepository;
    }

    /**
     * @Transactional: dirty checking handles update — no explicit save() needed.
     * Without @Transactional: entity is DETACHED → changes not persisted.
     */
    @Transactional
    public void updateStatus(Long id, ProjectStatus newStatus) {
        Project project = projectRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Project", id));
        project.setStatus(newStatus); // dirty check → UPDATE on flush
        // No save() needed!
    }

    /**
     * readOnly=true: skip dirty checking, no entity snapshot.
     * Better for queries — use consistently on read operations.
     */
    @Transactional(readOnly = true)
    public List<ProjectResponse> findAllWithOwner() {
        return projectRepository.findByStatusWithAll(ProjectStatus.ACTIVE)
            .stream()
            .map(ProjectResponse::from)
            .toList();
    }
}
```

```java
// test/ProjectRepositoryTest.java
@DataJpaTest   // loads JPA slice only: no web, no beans
@Testcontainers
class ProjectRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configure(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired UserRepository userRepository;
    @Autowired ProjectRepository projectRepository;

    @Test
    void should_incrementVersion_on_update() {
        User owner = userRepository.save(new User("test@example.com", "Test User"));
        Project project = projectRepository.save(new Project("Test", owner, null));
        assertThat(project.getVersion()).isEqualTo(0L);

        project.setStatus(ProjectStatus.ACTIVE);
        Project updated = projectRepository.saveAndFlush(project);
        assertThat(updated.getVersion()).isEqualTo(1L);
    }

    @Test
    void should_fetchJoin_produce_single_query() {
        // With show-sql=true: verify only 1 query in log
        User owner = userRepository.save(new User("owner@test.com", "Owner"));
        projectRepository.save(new Project("P1", owner, null));
        projectRepository.save(new Project("P2", owner, null));

        List<Project> projects = projectRepository.findByStatusWithAll(ProjectStatus.DRAFT);
        // Access owner and tasks — should NOT trigger additional queries
        projects.forEach(p -> {
            assertNotNull(p.getOwner().getName()); // would fail if separate query
        });
    }
}
```

---

## 4. Lab Steps

1. **N+1 detection:** `findAll()` + `p.getOwner().getName()` → count queries in log (show-sql=true)
2. **Fix N+1:** Switch to `findAllWithOwner()` → verify 1 query
3. **Dirty checking:** Update status without save() → verify DB updated (SELECT after commit)
4. **Optimistic lock:** Simulate concurrent update (two sessions, same version) → `OptimisticLockingFailureException`
5. **EXPLAIN ANALYZE:** Run `EXPLAIN ANALYZE SELECT * FROM tasks WHERE project_id=1 AND status='TODO'` before/after index
6. **Testcontainers:** Run test → verify real PostgreSQL container started (check Docker)

---

## 5. Checklist tự kiểm tra

- [ ] `show-sql=true` — đếm queries: 1 query là tốt, 1+N là xấu
- [ ] `@Transactional` method: update entity → không cần `save()` (dirty checking)
- [ ] `@Transactional(readOnly=true)` trên tất cả read methods
- [ ] `LazyInitializationException` → access lazy field outside transaction → fix với FETCH JOIN hoặc map to DTO inside transaction
- [ ] `@Version` → `OptimisticLockingFailureException` khi concurrent update → must retry
- [ ] `ddl-auto=validate` in production — Liquibase quản lý schema
- [ ] `DISTINCT` trong FETCH JOIN với collection để tránh duplicate rows
