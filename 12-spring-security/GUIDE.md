# Phase 12 — Spring Security · Lý thuyết & Lab

---

## 1. Lý thuyết

### 1.1 Security Filter Chain — Cơ chế hoạt động

**Spring Security là một chuỗi Servlet Filters.** Mỗi request phải đi qua chain trước khi đến Controller. Thứ tự quan trọng!

```
HTTP Request
    │
    ▼
┌──────────────────────────────────────────────────────┐
│  SecurityFilterChain (ordered filters)               │
│                                                       │
│  1. ChannelProcessingFilter (HTTPS redirect)         │
│  2. SecurityContextPersistenceFilter (load session)  │
│  3. CsrfFilter (CSRF token check)                    │
│  4. LogoutFilter                                     │
│  5. JwtAuthenticationFilter ← chúng ta thêm vào     │
│  6. UsernamePasswordAuthenticationFilter             │
│  7. ExceptionTranslationFilter (catch auth errors)   │
│  8. FilterSecurityInterceptor (authorization check)  │
└──────────────────────────────────────────────────────┘
    │
    ▼
  Controller (only if all filters pass)
```

**Authentication vs Authorization:**
- **Authentication (AuthN):** "Bạn là ai?" — verify identity. Kết quả: `Authentication` object trong `SecurityContextHolder`
- **Authorization (AuthZ):** "Bạn được làm gì?" — kiểm tra permissions. Xảy ra SAU authentication.

---

### 1.2 JWT — JSON Web Token

**Tại sao JWT thay vì Session:**
- **Session-based:** Server lưu state (session store) → horizontal scaling khó (phải share session giữa instances) hoặc sticky sessions
- **JWT-based:** Stateless — mọi thông tin trong token. Server chỉ verify chữ ký. Dễ scale horizontally.

**JWT Structure:**
```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyQHRlc3QuY29tIiwiZXhwIjoxNzIwMDAwMDAwfQ.abc123

Base64URL(Header) . Base64URL(Payload) . Signature
```

```json
// Header
{"alg": "HS256", "typ": "JWT"}

// Payload (claims)
{
  "sub": "user@test.com",   // subject
  "iat": 1719000000,        // issued at
  "exp": 1719900000,        // expiration
  "authorities": ["ROLE_USER", "project:read"]
}
// Signature = HMAC-SHA256(header + "." + payload, secretKey)
```

**Security considerations:**
- JWT payload visible nếu decoded (Base64, không encrypt). **Đừng lưu sensitive data** (password, credit card).
- Secret key phải đủ mạnh (>= 256 bits cho HS256).
- `exp` claim bắt buộc — JWT không expire là security risk.
- Access token: short-lived (15 minutes). Refresh token: long-lived (7 days), lưu HTTP-only cookie.

**Access + Refresh token flow:**
```
Client: POST /auth/login → Server: {accessToken (15min), refreshToken (7days)}
Client: GET /api/data    Authorization: Bearer <accessToken>
Client: accessToken expired → POST /auth/refresh Body: {refreshToken} → Server: new accessToken
Client: refreshToken expired → login again
```

---

### 1.3 `SecurityContextHolder` — Thread-local Storage

Spring Security lưu `Authentication` object trong `SecurityContextHolder` (ThreadLocal per request thread). Mọi code trong cùng thread có thể access:

```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName(); // email trong trường hợp của chúng ta
Collection<? extends GrantedAuthority> authorities = auth.getAuthorities();
```

**Stateless session (`SessionCreationPolicy.STATELESS`):**
SecurityContextHolder cleared sau mỗi request. Mỗi request phải re-authenticate từ JWT. Server không tạo HTTP session.

---

### 1.4 RBAC — Role-Based Access Control

**Flat permission model:**
```java
enum Permission {
    PROJECT_READ("project:read"),
    PROJECT_WRITE("project:write"),
    USER_MANAGE("user:manage")
}

enum Role {
    USER(Set.of(PROJECT_READ)),
    MANAGER(Set.of(PROJECT_READ, PROJECT_WRITE)),
    ADMIN(Set.of(Permission.values())) // all
}
```

**Authorities in Spring Security:**
Spring Security không phân biệt Role và Permission — tất cả là `GrantedAuthority`. Convention:
- Role: `ROLE_` prefix (required for `hasRole()`)
- Permission: no prefix (for `hasAuthority()`)

```java
// getAuthorities() returns BOTH role and permissions:
["ROLE_MANAGER", "project:read", "project:write"]

// In SecurityConfig:
.hasRole("MANAGER")          // checks "ROLE_MANAGER" (auto-adds ROLE_ prefix)
.hasAuthority("project:read") // checks exact string
```

**Method-level security (`@PreAuthorize`):**
```java
@PreAuthorize("hasAuthority('project:write')")         // simple
@PreAuthorize("hasRole('ADMIN')")                       // role
@PreAuthorize("authentication.name == #email")          // dynamic: own resource
@PreAuthorize("@projectService.isOwner(#id, authentication.name)") // custom bean method
```

---

### 1.5 `OncePerRequestFilter` — JWT Filter Pattern

**Tại sao `OncePerRequestFilter`:**
Đảm bảo filter chỉ chạy một lần per request, kể cả request dispatching (forward, include).

**JWT filter flow:**
```
Request → Extract "Authorization: Bearer <token>"
    → token null hoặc không phải Bearer? → pass to next filter (security config decides)
    → Extract username từ token (verify signature)
    → Load UserDetails từ DB bằng username
    → Validate token (not expired, username match)
    → Set Authentication trong SecurityContextHolder
    → Call chain.doFilter() → next filter
```

**Tại sao load UserDetails từ DB mỗi request:**
Để detect disabled/deleted users. JWT còn valid nhưng user bị ban → `enabled=false` trong UserDetails → authentication fail. Trade-off với performance: có thể cache UserDetails ngắn hạn.

---

## 2. Vấn đề thường gặp & Cách fix

### Issue 1: 403 khi `hasRole` vs `hasAuthority`

```java
// ❌ user có authority "ROLE_ADMIN" nhưng config dùng hasAuthority("ADMIN")
.requestMatchers("/admin/**").hasAuthority("ADMIN") // "ADMIN" khác "ROLE_ADMIN"!

// ✅ Nhất quán:
.requestMatchers("/admin/**").hasRole("ADMIN")     // hasRole tự thêm "ROLE_" prefix
// hoặc
.requestMatchers("/admin/**").hasAuthority("ROLE_ADMIN") // exact match
```

### Issue 2: CORS error khi Frontend gọi API

```java
// Với stateless JWT, cần configure CORS
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("http://localhost:3000", "https://yourdomain.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("*"));
    config.setAllowCredentials(true); // nếu dùng cookie
    
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}

// Trong SecurityFilterChain:
.cors(cors -> cors.configurationSource(corsConfigurationSource()))
```

### Issue 3: JWT Secret terlalu ngắn

```
io.jsonwebtoken.security.WeakKeyException: 
The specified key byte array is 64 bits which is not secure enough
```

**Fix:**
```java
// ❌ Secret quá ngắn
String secret = "mysecret"; // 7 chars = 56 bits < 256 bits required for HS256

// ✅ At least 256 bits (32 chars minimum, recommend 64+)
String secret = "this-is-a-very-long-secret-key-at-least-32-chars";

// Production: store in environment variable or secrets manager
@Value("${JWT_SECRET}") String secret;
```

---

## 3. Code mẫu

```java
// security/JwtService.java
@Service
public class JwtService {

    private final SecretKey secretKey;
    private final long accessTokenExpiryMs;

    public JwtService(
            @Value("${app.security.jwt-secret}") String secret,
            @Value("${app.security.access-token-expiry}") long expirySeconds) {
        this.secretKey = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.accessTokenExpiryMs = expirySeconds * 1000L;
    }

    public String generateToken(UserDetails user) {
        Map<String, Object> claims = new HashMap<>();
        // Store authorities in token — avoid DB lookup on every request
        claims.put("authorities", user.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .toList());

        return Jwts.builder()
            .claims(claims)
            .subject(user.getUsername())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + accessTokenExpiryMs))
            .signWith(secretKey)
            .compact();
    }

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public boolean isTokenValid(String token, UserDetails userDetails) {
        String username = extractUsername(token);
        return username.equals(userDetails.getUsername()) && !isExpired(token);
    }

    private boolean isExpired(String token) {
        return extractClaim(token, Claims::getExpiration).before(new Date());
    }

    private <T> T extractClaim(String token, Function<Claims, T> claimResolver) {
        return claimResolver.apply(
            Jwts.parser().verifyWith(secretKey).build()
                .parseSignedClaims(token).getPayload()
        );
    }
}
```

```java
// security/JwtAuthenticationFilter.java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    public JwtAuthenticationFilter(JwtService jwtService, UserDetailsService userDetailsService) {
        this.jwtService = jwtService;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String authHeader = request.getHeader("Authorization");

        // No token or wrong format → skip (let security config handle auth requirement)
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }

        String token = authHeader.substring(7); // Remove "Bearer "

        try {
            String username = jwtService.extractUsername(token);

            // Only set authentication if not already set (avoid re-processing)
            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                if (jwtService.isTokenValid(token, userDetails)) {
                    UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(
                            userDetails, null, userDetails.getAuthorities()
                        );
                    authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(authToken);
                }
            }
        } catch (Exception e) {
            // Invalid/expired token → log and continue (security config returns 401)
            logger.warn("JWT validation failed: " + e.getMessage());
        }

        chain.doFilter(request, response);
    }
}
```

```java
// security/SecurityConfig.java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity // enables @PreAuthorize
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtFilter;
    private final CustomAuthEntryPoint authEntryPoint;
    private final CustomAccessDeniedHandler accessDeniedHandler;

    public SecurityConfig(JwtAuthenticationFilter jwtFilter,
                          CustomAuthEntryPoint authEntryPoint,
                          CustomAccessDeniedHandler accessDeniedHandler) {
        this.jwtFilter = jwtFilter;
        this.authEntryPoint = authEntryPoint;
        this.accessDeniedHandler = accessDeniedHandler;
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())  // REST API — stateless, no CSRF needed
            .sessionManagement(sm -> sm
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)) // no HTTP session

            .authorizeHttpRequests(auth -> auth
                // Public endpoints
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/actuator/health").permitAll()

                // Role-based (role = coarse-grained)
                .requestMatchers(HttpMethod.DELETE, "/api/v1/**").hasRole("ADMIN")

                // Permission-based (permission = fine-grained)
                .requestMatchers(HttpMethod.POST, "/api/v1/projects").hasAuthority("project:write")
                .requestMatchers(HttpMethod.GET, "/api/v1/projects/**").hasAuthority("project:read")

                // Everything else: just needs to be authenticated
                .anyRequest().authenticated()
            )

            // Add JWT filter BEFORE the standard username/password filter
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)

            // Custom error responses (JSON, not HTML)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(authEntryPoint)  // 401
                .accessDeniedHandler(accessDeniedHandler)  // 403
            )
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12); // cost factor 12: ~300ms on modern CPU
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

```java
// security/CustomAuthEntryPoint.java — 401 JSON
@Component
public class CustomAuthEntryPoint implements AuthenticationEntryPoint {
    private final ObjectMapper objectMapper;

    public CustomAuthEntryPoint(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
                         AuthenticationException authException) throws IOException {
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.getWriter().write(objectMapper.writeValueAsString(
            Map.of("code", "UNAUTHORIZED",
                   "message", "Authentication required",
                   "path", request.getRequestURI())
        ));
    }
}

// security/CustomAccessDeniedHandler.java — 403 JSON
@Component
public class CustomAccessDeniedHandler implements AccessDeniedHandler {
    private final ObjectMapper objectMapper;

    public CustomAccessDeniedHandler(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response,
                       AccessDeniedException ex) throws IOException {
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.getWriter().write(objectMapper.writeValueAsString(
            Map.of("code", "FORBIDDEN",
                   "message", "Insufficient permissions",
                   "path", request.getRequestURI())
        ));
    }
}
```

---

## 4. Lab Steps

1. **Login flow:** `POST /auth/login` → get JWT → decode at jwt.io → xem claims
2. **Auth required:** `GET /api/v1/projects` without token → 401 JSON (không phải HTML!)
3. **Role enforcement:** Login as USER, try DELETE → 403. Login as ADMIN → 204
4. **Token expiry:** Set expiry to 5s → get token → wait 6s → use token → 401
5. **Disable user:** Login → get token → set `enabled=false` in DB → use token → verify still works (token valid) vs rethink with refresh token
6. **Method security:** `@PreAuthorize("@projectService.isOwner(#id, authentication.name)")` → test as non-owner

---

## 5. Checklist tự kiểm tra

- [ ] `hasRole("ADMIN")` auto-adds "ROLE_" prefix — `hasAuthority("ROLE_ADMIN")` là cùng meaning
- [ ] JWT filter: nếu không có token → `chain.doFilter()` — không throw 401 (security config quyết định)
- [ ] `SessionCreationPolicy.STATELESS` — mỗi request phải authenticate lại từ JWT
- [ ] `BCryptPasswordEncoder(12)` — không hardcode plain text password
- [ ] Custom 401/403 trả JSON, không phải Spring's default HTML white-label
- [ ] Access token: short (15min), Refresh token: long (7 days) và stored HTTP-only cookie
- [ ] Không lưu sensitive data trong JWT payload (visible sau Base64 decode)
