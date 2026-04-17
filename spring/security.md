# Spring Security

## 1. Core Architecture

```
Request → Filter Chain → Security Context → Controller
                ↓
    UsernamePasswordAuthenticationFilter
    BearerTokenAuthenticationFilter
    ExceptionTranslationFilter
    FilterSecurityInterceptor
```

### Security Filter Chain
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)          // disable for stateless APIs
            .sessionManagement(sm ->
                sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.GET, "/api/products/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(customEntryPoint)   // 401
                .accessDeniedHandler(customAccessDeniedHandler) // 403
            )
            .build();
    }
}
```

---

## 2. Authentication vs Authorization

| | Authentication | Authorization |
|---|---|---|
| Question | *Who are you?* | *What can you do?* |
| Spring class | `AuthenticationManager` | `AccessDecisionManager` / `AuthorizationManager` |
| Storage | `SecurityContextHolder` | `GrantedAuthority` list |
| HTTP status | 401 Unauthorized | 403 Forbidden |

---

## 3. UserDetailsService

```java
@Service
public class UserDetailsServiceImpl implements UserDetailsService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByEmail(username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));

        return org.springframework.security.core.userdetails.User.builder()
            .username(user.getEmail())
            .password(user.getPasswordHash())
            .roles(user.getRoles().stream()
                .map(Role::getName)
                .toArray(String[]::new))
            .accountExpired(!user.isActive())
            .credentialsExpired(user.isPasswordExpired())
            .disabled(!user.isEnabled())
            .build();
    }
}
```

### Password Encoding
```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12); // cost factor 12 (higher = slower = more secure)
    // Alternative: Argon2PasswordEncoder, SCryptPasswordEncoder
}

// Encoding
String hash = encoder.encode(rawPassword);

// Verification — constant-time comparison prevents timing attacks
encoder.matches(rawPassword, storedHash);
```

**Never use MD5 or SHA-1 for passwords** — they're fast (bad for passwords), no salt by default.

---

## 4. JWT Authentication

### JWT Structure
```
header.payload.signature

eyJhbGciOiJIUzI1NiJ9    ← Base64({"alg":"HS256"})
.
eyJzdWIiOiJ1c2VyQGV4YW1wbGUuY29tIiwicm9sZXMiOlsiVVNFUiJdLCJleHAiOjE2OTk5OTk5OTl9
  ← Base64({"sub":"user@example.com","roles":["USER"],"exp":1699999999})
.
HMACSHA256(base64Header + "." + base64Payload, secret)
```

### JWT Filter Implementation
```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws IOException, ServletException {

        String header = request.getHeader("Authorization");
        if (header == null || !header.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }

        String token = header.substring(7);
        try {
            Claims claims = jwtService.parseToken(token);
            String username = claims.getSubject();

            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                UsernamePasswordAuthenticationToken auth =
                    new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());
                auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));

                SecurityContextHolder.getContext().setAuthentication(auth);
            }
        } catch (JwtException e) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return;
        }

        chain.doFilter(request, response);
    }
}
```

### JWT Service
```java
@Service
public class JwtService {
    private static final long ACCESS_TOKEN_EXPIRY  = 15 * 60;          // 15 minutes
    private static final long REFRESH_TOKEN_EXPIRY = 7 * 24 * 60 * 60; // 7 days

    @Value("${jwt.secret}") private String secret;

    public String generateAccessToken(UserDetails userDetails) {
        return Jwts.builder()
            .subject(userDetails.getUsername())
            .claim("roles", userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority).toList())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + ACCESS_TOKEN_EXPIRY * 1000))
            .signWith(getSigningKey())
            .compact();
    }

    public Claims parseToken(String token) {
        return Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }

    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(Decoders.BASE64.decode(secret));
    }
}
```

### Token Refresh Strategy
```
Client             Server
  │── POST /auth/login ──────────────────→│
  │←─ { accessToken (15m), refreshToken (7d) }──│
  │                                        │
  │── GET /api/data (accessToken) ────────→│
  │←─ 200 OK ──────────────────────────────│
  │                                        │
  │  [access token expires]               │
  │── GET /api/data (expired token) ──────→│
  │←─ 401 Unauthorized ────────────────────│
  │                                        │
  │── POST /auth/refresh (refreshToken) ──→│
  │←─ { newAccessToken, newRefreshToken } ─│  (rotate refresh token!)
```

**Refresh token best practices:**
- Store refresh tokens in the DB (allows invalidation on logout/password change)
- Rotate refresh tokens on each use — detect token theft
- Use short-lived access tokens (5–15 minutes)
- Store access tokens in memory (not localStorage — XSS risk)

---

## 5. OAuth2

### Key Flows

#### Authorization Code Flow (for web apps)
```
User → App → Authorization Server (AS)
           → User logs in + consents
           ← Authorization Code
App → AS (code + client_secret) → Access Token
App → Resource Server (access token) → Protected Data
```

#### Client Credentials Flow (for services)
```
Service → AS (client_id + client_secret) → Access Token
Service → Resource Server (access token) → Data
// No user involved — machine-to-machine
```

### Spring Security OAuth2 Resource Server
```java
@Configuration
@EnableWebSecurity
public class ResourceServerConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .oauth2ResourceServer(oauth2 ->
                oauth2.jwt(jwt -> jwt.jwtAuthenticationConverter(jwtConverter())))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated())
            .build();
    }

    @Bean
    public JwtAuthenticationConverter jwtConverter() {
        JwtGrantedAuthoritiesConverter authoritiesConverter = new JwtGrantedAuthoritiesConverter();
        authoritiesConverter.setAuthoritiesClaimName("roles");
        authoritiesConverter.setAuthorityPrefix("ROLE_");

        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(authoritiesConverter);
        return converter;
    }
}
```

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com  # Spring fetches JWKS automatically
          # or:
          jwk-set-uri: https://auth.example.com/.well-known/jwks.json
```

---

## 6. Method-Level Security

```java
@Configuration
@EnableMethodSecurity // enables @PreAuthorize, @PostAuthorize, @Secured
public class MethodSecurityConfig { }

@Service
public class OrderService {

    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
    public List<Order> findOrdersByUser(Long userId) { ... }

    @PostAuthorize("returnObject.ownerId == authentication.principal.id")
    public Order findById(Long id) { ... }

    @PreAuthorize("hasRole('ADMIN')")
    @PostFilter("filterObject.status != 'DRAFT'")
    public List<Order> findAll() { ... }

    // SpEL expressions
    @PreAuthorize("@orderSecurityService.canAccess(#orderId, authentication)")
    public Order getOrder(Long orderId) { ... }
}

// Custom security bean for complex rules
@Component("orderSecurityService")
public class OrderSecurityService {
    public boolean canAccess(Long orderId, Authentication auth) {
        // complex logic
    }
}
```

---

## 7. CSRF and CORS

### CSRF
Cross-Site Request Forgery — malicious site tricks browser into making authenticated requests.

```java
// Disable for stateless JWT APIs (tokens not sent automatically by browser)
http.csrf(AbstractHttpConfigurer::disable)

// Enable for session-based (default) — Spring generates CSRF token
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
    // Cookie: XSRF-TOKEN; Header: X-XSRF-TOKEN required in requests
);
```

### CORS
```java
@Configuration
public class CorsConfig {

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOriginPatterns(List.of("https://*.example.com"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        config.setExposedHeaders(List.of("X-Total-Count"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}
```

---

## 8. Security Best Practices

```java
// 1. Never expose stack traces in error responses
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleException(Exception e) {
    log.error("Unhandled exception", e);
    return ResponseEntity.status(500).body(new ErrorResponse("Internal server error"));
    // Never: e.getMessage() directly to client — may expose internals
}

// 2. Rate limiting (with Bucket4j or Spring Security's built-in)
// 3. Audit logging for sensitive operations
// 4. Input validation at controller level
@PostMapping("/users")
public ResponseEntity<User> createUser(@RequestBody @Valid CreateUserRequest req) { ... }

// 5. Sensitive data not in URLs (visible in logs)
// Bad:  GET /api/users?password=secret
// Good: POST /api/auth/login { "password": "secret" }

// 6. HTTPS only — redirect HTTP to HTTPS
http.requiresChannel(channel -> channel.anyRequest().requiresSecure());
```

---

## 9. Common Interview Questions

| Question | Answer |
|----------|--------|
| What is the Spring Security filter chain and how is it structured? | Spring Security registers a `FilterChainProxy` as a single servlet filter. It holds a list of `SecurityFilterChain` instances, each matched by a request matcher. The matching chain contains an ordered list of `Filter` implementations (e.g., `UsernamePasswordAuthenticationFilter`, `BearerTokenAuthenticationFilter`, `ExceptionTranslationFilter`, `AuthorizationFilter`). Each filter handles one concern and calls the next. You configure the chain via `HttpSecurity` in a `@SecurityFilterChain` bean. Multiple chains allow different security rules for different URL patterns (e.g., `/api/**` vs `/admin/**`). |
| What is the difference between HTTP 401 and 403? | **401 Unauthorized** means the request lacks valid authentication credentials — the server does not know who you are. The response should include a `WWW-Authenticate` header prompting the client to authenticate. **403 Forbidden** means the server knows who you are (authenticated) but you do not have permission to access the resource (not authorized). Returning 401 when you mean 403, or vice versa, is a common security mistake that confuses clients. |
| How does JWT work for stateless authentication? | A JWT has three Base64URL-encoded parts separated by dots: **header** (algorithm, token type), **payload** (claims: subject, expiry, roles), **signature** (HMAC or RSA/EC over header+payload). The server signs the token on login and sends it to the client. On each request, the client sends the token in `Authorization: Bearer <token>`. The server verifies the signature with its secret/key — if valid, it trusts the claims without any database lookup. This makes authentication stateless — no session storage needed. |
| Why is storing JWT in `localStorage` a security risk? | Any JavaScript running on the page — including injected XSS payloads — can read `localStorage`. If an attacker injects a script, they can steal all JWTs and use them to impersonate users. Safer options: (1) **`httpOnly` cookies** — inaccessible to JavaScript; CSRF-protected with `SameSite=Strict/Lax` or a CSRF token; (2) **in-memory** (`sessionStorage` or a JS variable) — stolen only during an active XSS attack on the current session, not persisted. For APIs consumed by SPAs, `httpOnly` cookies with `SameSite` is the recommended approach. |
| What is the difference between `@PreAuthorize` and `@Secured`? | `@Secured({"ROLE_ADMIN"})` accepts a simple list of role names — no expressions. `@PreAuthorize("hasRole('ADMIN') and #userId == authentication.principal.id")` uses **Spring Expression Language (SpEL)**, enabling access to method arguments, the current `Authentication` object, and arbitrary conditions. `@PostAuthorize` is evaluated after the method returns and can filter based on the return value. `@PreAuthorize` is strictly more capable; `@Secured` is simpler but limited. Both require `@EnableMethodSecurity` (Boot 3) or `@EnableGlobalMethodSecurity`. |
| When should you disable CSRF protection and when must you keep it? | CSRF (Cross-Site Request Forgery) exploits the browser's automatic cookie-sending to forge authenticated requests. It is only relevant when authentication is **cookie/session based**. For stateless REST APIs using JWT in `Authorization` headers, CSRF is not a threat because cookies are not sent — disable it with `.csrf(AbstractHttpConfigurer::disable)` to avoid the overhead. **Keep CSRF enabled** for server-rendered applications or any endpoint where browsers automatically send authentication cookies. |
| What is OAuth2 and what are the main grant flows? | OAuth2 is an **authorization framework** that lets a user grant a third-party application limited access to their resources without sharing their credentials. Key flows: **Authorization Code** (for web/mobile apps — user is redirected to the auth server, gets a code, exchanges it for tokens); **Client Credentials** (machine-to-machine — no user involved, service authenticates directly with client ID/secret); **PKCE** (Proof Key for Code Exchange — Authorization Code variant for SPAs/mobile apps that cannot keep a secret). OAuth2 issues access tokens; OpenID Connect adds identity (ID tokens). |
| How does a Spring Boot resource server validate an incoming JWT? | The resource server fetches the authorization server's **JWKS** (JSON Web Key Set) endpoint to get the public keys. For each incoming request with a `Bearer` token: (1) decode the JWT header to find the key ID (`kid`); (2) find the matching public key in the JWKS; (3) verify the signature; (4) check `exp` (not expired), `iss` (correct issuer), and `aud` (correct audience). On success, the claims are extracted and wrapped in a `JwtAuthenticationToken` stored in `SecurityContextHolder`. Spring Boot configures this automatically with `spring.security.oauth2.resourceserver.jwt.jwk-set-uri`. |
| What is BCrypt and why is it suitable for password hashing? | BCrypt is a password hashing algorithm that incorporates a **random salt** (eliminating rainbow table attacks) and a configurable **cost factor** (work factor). Increasing the cost factor doubles the computation time, keeping brute-force attacks expensive as hardware improves. BCrypt outputs a fixed-size hash that includes the salt and cost factor — no need to store them separately. It is one-way: you cannot recover the password from the hash. Spring Security's `BCryptPasswordEncoder` defaults to cost 10; increase to 12+ for modern hardware. Never use MD5 or SHA-1 for passwords. |
| What is CORS and how do you configure it in Spring Boot? | CORS (Cross-Origin Resource Sharing) is a browser security mechanism that blocks a web page from making requests to a different origin (domain/port/protocol) than the one that served it. The browser sends a preflight `OPTIONS` request; the server must respond with `Access-Control-Allow-Origin` and related headers. In Spring Boot: use `@CrossOrigin` on a controller, define a `CorsConfiguration` bean, or configure it in `HttpSecurity.cors()` with a `CorsConfigurationSource`. Always specify allowed origins explicitly — never use wildcard `*` in production for credentialed requests. |
