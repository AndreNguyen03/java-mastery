# Phase 10 — Spring Boot 3 · Lý thuyết & Lab

---

## 1. Lý thuyết

### 1.1 Auto-configuration — Magic Explained

**Vấn đề trước Spring Boot:** Mỗi Spring project phải manually configure DispatcherServlet, Jackson, DataSource, TransactionManager, etc. Hàng chục `@Bean` methods chỉ để "bật" framework.

**Auto-configuration cơ chế:**
1. `@SpringBootApplication` includes `@EnableAutoConfiguration`
2. Spring Boot scan `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` trong tất cả JARs
3. Mỗi `@AutoConfiguration` class có `@ConditionalOn...` conditions
4. Conditions được evaluate: nếu thoả mãn → auto-config được applied

```
spring-boot-autoconfigure.jar
└── META-INF/spring/...AutoConfiguration.imports
    ├── DataSourceAutoConfiguration
    │   @ConditionalOnClass(DataSource.class)  ← H2 hoặc PostgreSQL jar có mặt
    │   @ConditionalOnMissingBean(DataSource)   ← bạn chưa define DataSource bean
    │   → auto-creates DataSource từ application.properties
    ├── JacksonAutoConfiguration
    ├── HibernateJpaAutoConfiguration
    └── ...100+ configs
```

**`@Conditional` conditions:**
- `@ConditionalOnClass(X.class)` — X phải có trong classpath
- `@ConditionalOnMissingBean(X.class)` — bạn chưa define X bean (backoff)
- `@ConditionalOnProperty("spring.x.enabled=true")` — property phải true
- `@ConditionalOnWebApplication` — chỉ trong web context

**Override auto-config:** Define your own `@Bean` → auto-config backs off (`@ConditionalOnMissingBean`).

---

### 1.2 Request Lifecycle — Từ HTTP đến Controller

```
Browser → [Server Socket] → [Tomcat Thread Pool]
    → [Filter Chain] (CorrelationIdFilter, SecurityFilter, ...)
    → [DispatcherServlet.doDispatch()]
        → [HandlerMapping] → find @RequestMapping method
        → [HandlerInterceptor.preHandle()] (e.g. AuthInterceptor)
        → [ArgumentResolver] → resolve @RequestBody, @PathVariable, @RequestParam
        → [Validator] → @Valid → MethodArgumentNotValidException if fail
        → [Controller method] → your code
        → [HandlerInterceptor.postHandle()]
        → [MessageConverter] → serialize return value to JSON
        → [ExceptionHandler] if exception thrown
    → [Filter Chain] (reverse order, response filters)
    → [HTTP Response to Browser]
```

**Filter vs Interceptor vs ArgumentResolver:**
| | Filter | Interceptor | ArgumentResolver |
|---|---|---|---|
| Scope | Servlet-level | Spring MVC-level | Method parameter |
| Access | Request/Response | Request/Response + Handler | Parameter type |
| Use for | Auth, logging, compression | Auth check, logging | Custom parameter binding |
| Exception | Catch in filter | Use `@ControllerAdvice` | Return null or throw |

---

### 1.3 DTO Pattern — Tại sao tách DTO khỏi Entity

**Vấn đề expose Entity trực tiếp:**
1. Expose internal structure → coupling giữa API contract và DB schema
2. Bidirectional relationships → infinite recursion khi serialize (Jackson)
3. Sensitive fields (`password`, `internalStatus`) bị expose
4. Validation rules API ≠ validation rules DB

```
Request → [CreateProjectRequest DTO] → [Controller] → [Service] → [Project Entity] → DB
                 (API contract)                                     (DB model)
DB → [Project Entity] → [Service] → [ProjectResponse DTO] → [Controller] → Response
                                          (what client sees)
```

**DTO immutability với record:**
```java
// Validate at boundary — fail fast
public record CreateProjectRequest(
    @NotBlank(message = "Name required") String name,
    @Size(max = 500) String description,
    @Future(message = "Deadline must be in the future") LocalDate deadline
) {}
```

---

### 1.4 Exception Handling — `@RestControllerAdvice`

**Tại sao không dùng try-catch trong controller:**
- Controller nên chỉ deal with HTTP concerns (parse request, return response)
- Exception handling scattered khắp controllers → duplicate code
- Khó ensure consistent error format

**`@RestControllerAdvice` = global exception handler:**
```java
@RestControllerAdvice
@Slf4j
class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    ErrorResponse handleNotFound(ResourceNotFoundException ex) {
        return new ErrorResponse("NOT_FOUND", ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(FieldError::getField, FieldError::getDefaultMessage));
        return new ErrorResponse("VALIDATION_FAILED", errors);
    }

    @ExceptionHandler(Exception.class) // catch-all
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    ErrorResponse handleGeneric(Exception ex, HttpServletRequest request) {
        log.error("Unhandled exception for {}", request.getRequestURI(), ex);
        return new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred");
        // NOTE: Don't expose ex.getMessage() to client — may contain sensitive info
    }
}
```

---

### 1.5 Pagination — Tại sao cần

**Vấn đề không có pagination:**
`SELECT * FROM projects` trả 1 triệu rows → serialize JSON → response 500MB → OOM hoặc timeout.

**Spring Data Pageable:**
```java
// Controller
@GetMapping
Page<ProjectResponse> list(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size,
    @RequestParam(defaultValue = "createdAt") String sort
) {
    Pageable pageable = PageRequest.of(page, size, Sort.by(sort).descending());
    return projectService.list(pageable).map(ProjectResponse::from);
}

// Response includes pagination metadata:
{
    "content": [...],
    "page": { "number": 0, "size": 20, "totalElements": 1250, "totalPages": 63 }
}
```

**Cursor-based vs Offset pagination:**
- Offset (`LIMIT 20 OFFSET 200`): dễ implement, nhưng inconsistent khi data đổi, O(OFFSET) DB scan
- Cursor-based (WHERE created_at < cursor): consistent, O(1), nhưng không biết tổng số pages

---

## 2. Vấn đề thường gặp & Cách fix

### Issue 1: `@Valid` không hoạt động

```java
// ❌ Missing @Valid annotation — Spring không trigger validation
@PostMapping
public ResponseEntity<...> create(@RequestBody CreateProjectRequest req) { ... }

// ✅ @Valid triggers constraint validation
@PostMapping
public ResponseEntity<...> create(@RequestBody @Valid CreateProjectRequest req) { ... }
// MethodArgumentNotValidException thrown if invalid → caught by @RestControllerAdvice
```

### Issue 2: Circular reference trong JSON serialization

```java
// ❌ Project có List<Task>, Task có Project → infinite recursion
@Entity class Project {
    @OneToMany List<Task> tasks;
}
@Entity class Task {
    @ManyToOne Project project; // ← back reference!
}
// → StackOverflowError khi serialize

// ✅ Dùng DTO — DTO không có circular reference
record ProjectResponse(Long id, String name, List<TaskSummary> tasks) {
    record TaskSummary(Long id, String title) {} // chỉ basic fields, không back-ref
}
```

### Issue 3: N+1 trong Controller (lazy load)

```java
// ❌ Controller serialize project.getTasks() → N+1 queries
@GetMapping
List<ProjectResponse> list() {
    return projectRepo.findAll().stream()
        .map(p -> new ProjectResponse(p.getId(), p.getName(), p.getTasks())) // ← lazy load!
        .toList();
}

// ✅ Fetch join in repository, map to DTO eagerly
@GetMapping
List<ProjectResponse> list() {
    return projectRepo.findAllWithTasks() // FETCH JOIN in JPQL
        .stream().map(ProjectResponse::from).toList();
}
```

---

## 3. Code mẫu

```java
// domain/Project.java
public class Project {
    private Long id;
    private String name;
    private String description;
    private ProjectStatus status;
    private LocalDate deadline;
    private LocalDateTime createdAt;

    public Project(String name, String description, LocalDate deadline) {
        this.name = name;
        this.description = description;
        this.deadline = deadline;
        this.status = ProjectStatus.PLANNING;
        this.createdAt = LocalDateTime.now();
    }

    /**
     * Domain logic: state transition rules in the domain object itself, not service.
     * Business invariant: PLANNING → ACTIVE → COMPLETED (can't skip steps)
     */
    public void updateStatus(ProjectStatus newStatus) {
        boolean valid = switch (this.status) {
            case PLANNING -> newStatus == ProjectStatus.ACTIVE || newStatus == ProjectStatus.CANCELLED;
            case ACTIVE -> newStatus == ProjectStatus.COMPLETED || newStatus == ProjectStatus.CANCELLED;
            case COMPLETED, CANCELLED -> false; // terminal states
        };
        if (!valid) throw new IllegalStateException(
            "Cannot transition from " + this.status + " to " + newStatus
        );
        this.status = newStatus;
    }

    // getters...
}

public enum ProjectStatus { PLANNING, ACTIVE, COMPLETED, CANCELLED }
```

```java
// dto/CreateProjectRequest.java
public record CreateProjectRequest(
    @NotBlank(message = "Project name is required")
    @Size(min = 3, max = 100, message = "Name must be 3-100 characters")
    String name,

    @Size(max = 500, message = "Description max 500 characters")
    String description,

    @Future(message = "Deadline must be in the future")
    LocalDate deadline
) {}

// dto/ProjectResponse.java
public record ProjectResponse(
    Long id,
    String name,
    String description,
    ProjectStatus status,
    LocalDate deadline,
    LocalDateTime createdAt
) {
    public static ProjectResponse from(Project project) {
        return new ProjectResponse(
            project.getId(), project.getName(), project.getDescription(),
            project.getStatus(), project.getDeadline(), project.getCreatedAt()
        );
    }
}
```

```java
// controller/ProjectController.java
@RestController
@RequestMapping("/api/v1/projects")
public class ProjectController {

    private static final Logger log = LoggerFactory.getLogger(ProjectController.class);
    private final ProjectService service;

    public ProjectController(ProjectService service) {
        this.service = service;
    }

    @PostMapping
    public ResponseEntity<ProjectResponse> create(@RequestBody @Valid CreateProjectRequest req) {
        Project created = service.create(req);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}").buildAndExpand(created.getId()).toUri();
        return ResponseEntity.created(location).body(ProjectResponse.from(created));
    }

    @GetMapping("/{id}")
    public ProjectResponse get(@PathVariable Long id) {
        return ProjectResponse.from(service.getById(id));
    }

    @GetMapping
    public Page<ProjectResponse> list(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") @Max(100) int size
    ) {
        return service.list(PageRequest.of(page, size)).map(ProjectResponse::from);
    }

    @PatchMapping("/{id}/status")
    public ProjectResponse updateStatus(
        @PathVariable Long id,
        @RequestBody @Valid UpdateStatusRequest req
    ) {
        return ProjectResponse.from(service.updateStatus(id, req.status()));
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        service.delete(id);
    }
}
```

```java
// web/GlobalExceptionHandler.java
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    record ErrorResponse(String code, Object message, Instant timestamp) {
        ErrorResponse(String code, Object message) { this(code, message, Instant.now()); }
    }

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    ErrorResponse handleNotFound(ResourceNotFoundException ex) {
        return new ErrorResponse("NOT_FOUND", ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> fieldErrors = ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                e -> e.getDefaultMessage() != null ? e.getDefaultMessage() : "Invalid",
                (e1, e2) -> e1 // keep first error per field
            ));
        return new ErrorResponse("VALIDATION_FAILED", fieldErrors);
    }

    @ExceptionHandler(IllegalStateException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    ErrorResponse handleConflict(IllegalStateException ex) {
        return new ErrorResponse("CONFLICT", ex.getMessage());
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    ErrorResponse handleGeneric(Exception ex, HttpServletRequest request) {
        log.error("Unhandled exception: {} {}", request.getMethod(), request.getRequestURI(), ex);
        return new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred");
    }
}
```

```java
// web/CorrelationIdFilter.java
@Component
@Order(1) // run first in filter chain
public class CorrelationIdFilter extends OncePerRequestFilter {

    private static final String HEADER = "X-Correlation-ID";

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String correlationId = req.getHeader(HEADER);
        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }

        MDC.put("correlationId", correlationId);
        res.setHeader(HEADER, correlationId); // echo back to client

        try {
            chain.doFilter(req, res);
        } finally {
            MDC.clear(); // MUST clear — thread pool reuse
        }
    }
}
```

```java
// test/ProjectControllerTest.java
@WebMvcTest(ProjectController.class) // only loads web layer — no DB
class ProjectControllerTest {

    @Autowired MockMvc mockMvc;
    @Autowired ObjectMapper objectMapper;
    @MockBean ProjectService service; // mock service

    @Test
    void should_createProject_and_return_201() throws Exception {
        Project created = new Project("Test", "desc", LocalDate.now().plusDays(30));
        when(service.create(any())).thenReturn(created);

        mockMvc.perform(post("/api/v1/projects")
            .contentType(MediaType.APPLICATION_JSON)
            .content(objectMapper.writeValueAsString(
                new CreateProjectRequest("Test", "desc", LocalDate.now().plusDays(30))
            )))
            .andExpect(status().isCreated())
            .andExpect(header().exists("Location"))
            .andExpect(jsonPath("$.name").value("Test"));
    }

    @Test
    void should_return_400_when_nameMissing() throws Exception {
        mockMvc.perform(post("/api/v1/projects")
            .contentType(MediaType.APPLICATION_JSON)
            .content("{\"description\":\"no name\"}"))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.code").value("VALIDATION_FAILED"))
            .andExpect(jsonPath("$.message.name").exists());
    }

    @Test
    void should_return_404_when_projectNotFound() throws Exception {
        when(service.getById(99L)).thenThrow(new ResourceNotFoundException("Project", 99L));

        mockMvc.perform(get("/api/v1/projects/99"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.code").value("NOT_FOUND"));
    }
}
```

---

## 4. Lab Steps

1. **Auto-config debug:** Add `--debug` flag → xem "CONDITIONS EVALUATION REPORT" — thấy DataSourceAutoConfiguration conditions
2. **Filter order:** Add 2 filters với `@Order(1)` và `@Order(2)`, print request → verify order
3. **Validation:** POST request thiếu `name` → verify 400 với field error trong response
4. **Pagination:** GET `/api/v1/projects?page=0&size=5&sort=name` → verify response structure
5. **ControllerAdvice:** Throw `ResourceNotFoundException` from service → verify 404 JSON format
6. **MockMvc:** Test `@WebMvcTest` — không start full context, nhanh hơn integration test

---

## 5. Checklist tự kiểm tra

- [ ] `ResponseEntity.created(location)` → HTTP 201 với `Location` header
- [ ] `@Valid` trên `@RequestBody` — thiếu annotation → validation không chạy
- [ ] `@RestControllerAdvice` catches exceptions globally — không cần try-catch trong controller
- [ ] DTO record: validate tại boundary (`@NotBlank`, `@Future`) thay vì trong service
- [ ] `@WebMvcTest` chỉ load web layer — nhanh, cần `@MockBean` cho dependencies
- [ ] CorrelationId filter: `MDC.clear()` trong `finally` — không thể bỏ
