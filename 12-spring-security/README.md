# Phase 12 — Spring Security

> **Mục tiêu:** Hiểu request đi qua Security Filter Chain như thế nào — từ đó debug được 401/403 mà không phải đoán mò.

---

## Security Filter Chain — Full Picture

```
HTTP Request
    ↓
SecurityContextPersistenceFilter   ← load SecurityContext từ session/token
    ↓
UsernamePasswordAuthenticationFilter / JwtAuthenticationFilter
    ↓ (extract credentials → authenticate)
AuthenticationManager
    ↓
AuthenticationProvider (DaoAuthenticationProvider, etc.)
    ↓
UserDetailsService.loadUserByUsername()
    ↓
SecurityContext.setAuthentication(authenticated)
    ↓
AuthorizationFilter (kiểm tra permission)
    ↓
Controller
```

---

## Part 1 — Authentication vs Authorization

| | Authentication | Authorization |
|--|---------------|---------------|
| **Là gì?** | Xác minh danh tính (bạn là ai) | Kiểm tra quyền (bạn được làm gì) |
| **Spring** | `AuthenticationManager`, `UserDetailsService` | `AuthorizationManager`, `@PreAuthorize` |
| **Thất bại** | 401 Unauthorized | 403 Forbidden |

---

## Part 2 — SecurityFilterChain

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())          // REST API không cần CSRF
            .sessionManagement(sm -> sm.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/projects/**").hasRole("USER")
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(customEntryPoint)    // 401 handler
                .accessDeniedHandler(customAccessDeniedHandler) // 403 handler
            )
            .build();
    }
}
```

---

## Part 3 — JWT

### Token structure
```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjMiLCJyb2xlcyI6WyJST0xFX1VTRVIiXX0.signature
       Header                        Payload                           Signature
```

### JWT Filter
```java
@Component
class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                     FilterChain chain) throws IOException, ServletException {
        String header = req.getHeader("Authorization");
        if (header == null || !header.startsWith("Bearer ")) {
            chain.doFilter(req, res);
            return;
        }

        String token = header.substring(7);
        if (jwtService.isValid(token)) {
            String userId = jwtService.extractSubject(token);
            UserDetails user = userDetailsService.loadUserByUsername(userId);
            var auth = new UsernamePasswordAuthenticationToken(user, null, user.getAuthorities());
            auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(req));
            SecurityContextHolder.getContext().setAuthentication(auth);
        }

        chain.doFilter(req, res);
    }
}
```

### Access Token + Refresh Token
```
Login
    ↓
Server trả về:
  - access_token (short-lived: 15 phút)
  - refresh_token (long-lived: 7 ngày, stored in DB)

Access protected resource:
  → send access_token in Authorization header

Access token expired:
  → POST /auth/refresh với refresh_token
  → Server validate refresh_token trong DB
  → Trả về access_token mới (+ rotate refresh_token)

Logout:
  → Xoá refresh_token khỏi DB (invalidate)
```

---

## Part 4 — RBAC (Role-Based Access Control)

```java
// Roles: ADMIN, MANAGER, USER
// Permissions: project:read, project:write, task:read, task:write

enum Permission {
    PROJECT_READ("project:read"),
    PROJECT_WRITE("project:write"),
    TASK_READ("task:read"),
    TASK_WRITE("task:write");
}

enum Role {
    ADMIN(Set.of(PROJECT_READ, PROJECT_WRITE, TASK_READ, TASK_WRITE)),
    MANAGER(Set.of(PROJECT_READ, TASK_READ, TASK_WRITE)),
    USER(Set.of(PROJECT_READ, TASK_READ));
}
```

### Method Security
```java
@PreAuthorize("hasRole('ADMIN')")
void deleteProject(Long id) { ... }

@PreAuthorize("hasAuthority('project:write')")
void createProject(CreateProjectRequest req) { ... }

@PreAuthorize("@projectSecurityService.isOwner(authentication, #id)")
void updateProject(Long id, UpdateProjectRequest req) { ... }
```

---

## Part 5 — Security Headers

```java
http.headers(headers -> headers
    .frameOptions(fo -> fo.deny())                     // clickjacking
    .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'"))
    .httpStrictTransportSecurity(hsts -> hsts.maxAgeInSeconds(31536000))
    .xssProtection(xss -> xss.block(true))
);
```

### CORS
```java
@Bean
CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.example.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setAllowCredentials(true);
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}
```

---

## Part 6 — Cookie Security

```java
ResponseCookie cookie = ResponseCookie.from("refresh_token", token)
    .httpOnly(true)    // JavaScript không đọc được → XSS mitigation
    .secure(true)      // chỉ HTTPS
    .sameSite("Strict") // CSRF mitigation
    .path("/api/auth")
    .maxAge(Duration.ofDays(7))
    .build();
response.addHeader(HttpHeaders.SET_COOKIE, cookie.toString());
```

---

## Project — Identity & Authorization System

```
src/
├── domain/
│   ├── auth/
│   │   ├── AuthController.java      ← /register, /login, /refresh, /logout
│   │   ├── AuthService.java
│   │   └── dto/
│   │       ├── LoginRequest.java
│   │       ├── RegisterRequest.java
│   │       └── TokenResponse.java
│   └── user/
│       ├── User.java               ← @Entity with Role
│       ├── Role.java               (Enum)
│       ├── Permission.java         (Enum)
│       └── RefreshToken.java       ← @Entity (stored in DB)
├── security/
│   ├── SecurityConfig.java
│   ├── JwtService.java             ← generate, validate, extract
│   ├── JwtAuthenticationFilter.java
│   ├── UserDetailsServiceImpl.java
│   ├── CustomAuthEntryPoint.java   ← 401 JSON response
│   └── CustomAccessDeniedHandler.java ← 403 JSON response
└── config/
    └── PasswordEncoderConfig.java  ← BCryptPasswordEncoder
```

**Security scenarios để test:**
1. Token expired → 401 với descriptive message
2. Token valid nhưng không có quyền → 403
3. Refresh token revoked (user logout) → 401
4. CORS từ domain không được phép → blocked
5. Brute force login → rate limiting (optional: Redis counter)

---

## Checklist — 6 câu hỏi

Ví dụ với `JWT`:

| # | Câu hỏi | Trả lời |
|---|---------|---------|
| 1 | Là gì? | Self-contained token: base64(header).base64(payload).signature |
| 2 | Giải quyết gì? | Stateless authentication — server không cần store session |
| 3 | Hoạt động thế nào? | Server sign bằng secret; client gửi kèm mỗi request; server verify signature |
| 4 | Khi nào dùng? | Stateless REST API, microservices |
| 5 | Khi nào không? | Cần revoke ngay lập tức → JWT không revoke được trước khi expire → cần blacklist |
| 6 | Debug thế nào? | Decode tại jwt.io; check exp claim; verify `iss` và `aud` claims |

---

## Run

```bash
docker compose up -d
mvn spring-boot:run -pl phase-12-spring-security
mvn test -pl phase-12-spring-security
```
