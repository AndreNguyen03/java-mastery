# Phase 10 — Spring Boot

> **Mục tiêu:** Hiểu toàn bộ request lifecycle trong Spring MVC — từ lúc HTTP request đến network interface đến lúc JSON response rời server.

---

## Request Lifecycle Overview

```
Client HTTP Request
    ↓
Embedded Tomcat (HTTP connector)
    ↓
Filter Chain (Servlet Filters)
    ↓
DispatcherServlet
    ↓
HandlerMapping (find Controller method)
    ↓
HandlerInterceptor.preHandle()
    ↓
ArgumentResolver (deserialize request)
    ↓
Controller method execution
    ↓
Service → Repository (business logic)
    ↓
MessageConverter (serialize response)
    ↓
HandlerInterceptor.postHandle()
    ↓
ExceptionHandler (nếu có exception)
    ↓
HandlerInterceptor.afterCompletion()
    ↓
HTTP Response
```

---

## Part 1 — Auto-configuration

### Tại sao Spring Boot "just works"
```java
@SpringBootApplication
// = @Configuration + @ComponentScan + @EnableAutoConfiguration

// @EnableAutoConfiguration:
// Scan META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
// Conditionally register beans dựa trên classpath
```

```java
// Ví dụ: DataSourceAutoConfiguration
@ConditionalOnClass(DataSource.class)          // chỉ khi có JDBC trên classpath
@ConditionalOnMissingBean(DataSource.class)    // chỉ khi chưa có DataSource bean
@Configuration
class DataSourceAutoConfiguration {
    @Bean DataSource dataSource(DataSourceProperties props) { ... }
}
```

---

## Part 2 — Spring MVC

### Controller Layer
```java
@RestController
@RequestMapping("/api/v1/projects")
@RequiredArgsConstructor
class ProjectController {

    private final ProjectService projectService;

    @GetMapping
    ResponseEntity<Page<ProjectResponse>> findAll(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(required = false) String status) {
        return ResponseEntity.ok(projectService.findAll(page, size, status));
    }

    @PostMapping
    ResponseEntity<ProjectResponse> create(@RequestBody @Valid CreateProjectRequest req) {
        ProjectResponse created = projectService.create(req);
        URI location = URI.create("/api/v1/projects/" + created.id());
        return ResponseEntity.created(location).body(created);
    }

    @PatchMapping("/{id}/status")
    ResponseEntity<ProjectResponse> updateStatus(
            @PathVariable Long id,
            @RequestBody @Valid UpdateStatusRequest req) {
        return ResponseEntity.ok(projectService.updateStatus(id, req));
    }
}
```

### DTO Pattern
```java
// Request DTO — input validation
record CreateProjectRequest(
    @NotBlank String name,
    @NotNull Long ownerId,
    @Future LocalDate deadline
) {}

// Response DTO — output shape
record ProjectResponse(
    Long id,
    String name,
    String status,
    LocalDate deadline,
    UserSummary owner
) {}
```

**Tại sao DTO:** Tách contract từ domain entity. Entity có thể thay đổi schema DB mà không break API.

---

## Part 3 — Validation

```java
@NotNull         // not null
@NotBlank        // not null, not empty, not whitespace (String)
@Size(min=2, max=100)
@Min(0) @Max(999)
@Email
@Pattern(regexp = "^[A-Z]{3}$")
@Future @Past @PastOrPresent
@Valid           // trigger nested validation

// Custom validator
@Constraint(validatedBy = UniqueEmailValidator.class)
@interface UniqueEmail { String message() default "Email already exists"; ... }
```

---

## Part 4 — Exception Handling

```java
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(404).body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<FieldError> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> new FieldError(e.getField(), e.getDefaultMessage()))
            .toList();
        return ResponseEntity.badRequest().body(new ValidationErrorResponse(errors));
    }

    @ExceptionHandler(Exception.class)
    ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        log.error("Unexpected error", ex);
        return ResponseEntity.internalServerError().body(new ErrorResponse("INTERNAL_ERROR", "Something went wrong"));
    }
}
```

---

## Part 5 — Filter vs Interceptor vs ArgumentResolver

| Component | Scope | Dùng khi |
|-----------|-------|---------|
| **Filter** | Servlet level — trước Spring | Logging, authentication, CORS, rate limiting |
| **Interceptor** | Spring MVC level — sau DispatcherServlet | Auth check, audit logging, MDC |
| **ArgumentResolver** | Resolve method parameters | Custom annotation → inject current user |

```java
// ArgumentResolver — @CurrentUser inject User vào method param
@Component
class CurrentUserArgumentResolver implements HandlerMethodArgumentResolver {
    public boolean supportsParameter(MethodParameter param) {
        return param.hasParameterAnnotation(CurrentUser.class);
    }
    public Object resolveArgument(...) {
        return SecurityContextHolder.getContext().getAuthentication().getPrincipal();
    }
}

// Dùng trong Controller
@GetMapping("/me")
ResponseEntity<UserResponse> getMe(@CurrentUser User user) { ... }
```

---

## Part 6 — Pagination & Sorting

```java
@GetMapping
ResponseEntity<PagedResponse<TaskResponse>> getTasks(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "createdAt") String sortBy,
        @RequestParam(defaultValue = "desc") String direction) {

    Sort sort = direction.equals("asc") ? Sort.by(sortBy).ascending()
                                        : Sort.by(sortBy).descending();
    Pageable pageable = PageRequest.of(page, size, sort);
    Page<Task> tasks = taskService.findAll(pageable);
    return ResponseEntity.ok(PagedResponse.from(tasks));
}
```

---

## Part 7 — Actuator

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, env
  endpoint:
    health:
      show-details: always
```

```
GET /actuator/health       ← UP/DOWN + component details
GET /actuator/metrics      ← list available metrics
GET /actuator/metrics/jvm.memory.used ← specific metric
GET /actuator/env          ← environment properties
```

---

## Project — Project Management System

```
src/
├── domain/
│   ├── user/
│   │   ├── User.java (Entity)
│   │   ├── UserController.java
│   │   ├── UserService.java
│   │   ├── UserRepository.java (in-memory phase này)
│   │   ├── dto/
│   │   │   ├── CreateUserRequest.java   (Record)
│   │   │   └── UserResponse.java        (Record)
│   ├── project/
│   │   ├── Project.java
│   │   ├── ProjectStatus.java (Enum)
│   │   ├── ProjectController.java
│   │   ├── ProjectService.java
│   │   └── dto/...
│   └── task/
│       ├── Task.java
│       ├── TaskPriority.java (Enum)
│       ├── TaskController.java
│       ├── TaskService.java
│       └── dto/...
├── common/
│   ├── exception/
│   │   ├── ResourceNotFoundException.java
│   │   ├── ConflictException.java
│   │   └── GlobalExceptionHandler.java
│   ├── dto/
│   │   ├── ErrorResponse.java  (Record)
│   │   └── PagedResponse.java  (Record)
│   └── web/
│       ├── CorrelationIdFilter.java   ← MDC logging
│       └── RequestLoggingInterceptor.java
└── ProjectManagementApplication.java
```

**API Endpoints:**
```
POST   /api/v1/users
GET    /api/v1/users/{id}
POST   /api/v1/projects
GET    /api/v1/projects?status=ACTIVE&page=0&size=20
PATCH  /api/v1/projects/{id}/status
POST   /api/v1/projects/{projectId}/tasks
GET    /api/v1/projects/{projectId}/tasks?priority=HIGH
PATCH  /api/v1/tasks/{id}
```

**Testing:**
```java
@WebMvcTest(ProjectController.class)  // only web layer
class ProjectControllerTest {
    @MockBean ProjectService service;

    @Test
    void createProject_returns201() throws Exception {
        mockMvc.perform(post("/api/v1/projects")
            .contentType(APPLICATION_JSON)
            .content("""{"name":"Alpha","ownerId":1}"""))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.name").value("Alpha"));
    }
}
```

---

## Checklist — 6 câu hỏi

Ví dụ với `DispatcherServlet`:

| # | Câu hỏi | Trả lời |
|---|---------|---------|
| 1 | Là gì? | Front controller — single entry point cho tất cả HTTP requests |
| 2 | Giải quyết gì? | Central coordination: routing, argument resolving, exception handling |
| 3 | Hoạt động thế nào? | HandlerMapping tìm controller; invoke → serialize response |
| 4 | Khi nào dùng? | Always — bạn không control, Spring manage |
| 5 | Khi nào bypass? | Filters (trước DS), Static resources (ResourceHandler) |
| 6 | Debug thế nào? | `logging.level.org.springframework.web=DEBUG` → log routing decisions |

---

## Run

```bash
mvn spring-boot:run -pl phase-10-spring-boot
mvn test -pl phase-10-spring-boot
```
