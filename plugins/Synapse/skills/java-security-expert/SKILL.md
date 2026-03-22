---
name: java-security-expert
description: Expert guidance on Java application security, Spring Boot, Spring Security, modular architecture, Factory pattern, and system design. Use when writing secure Spring Boot services, implementing authentication/authorization (OAuth2, JWT, RBAC), designing factory-based components, reviewing code for vulnerabilities (OWASP Top 10), applying system design patterns, or when the user mentions Spring Security, JWT, OAuth2, CORS, CSRF, SQL injection, XSS, input validation, secure API design, factory pattern, service modules, or system design.
---

# Java Security Expert Skill

Applies rigorous security principles, Spring Boot best practices, Factory Pattern design, and modular system architecture to every piece of Java code.

## Core Philosophy

- **Security is not a feature — it is a constraint.** Every layer of the stack enforces it.
- **Modular by default.** No god classes. No god services. Each module owns exactly one concern.
- **Factory Pattern is the construction mechanism.** Objects are never `new`'d ad hoc in business logic.
- **Fail closed, not open.** When in doubt, deny access, reject input, throw a checked exception.

---

## 1. Spring Security Architecture

### Authentication vs Authorization — always separate concerns

```java
// Authentication: who are you?
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .csrf(AbstractHttpConfigurer::disable)           // stateless REST — disable CSRF
        .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/public/**").permitAll()
            .requestMatchers("/api/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated()
        )
        .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
    return http.build();
}
```

### JWT Filter — stateless token validation

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenService tokenService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String token = extractBearerToken(req);
        if (token != null && tokenService.isValid(token)) {
            String username = tokenService.extractUsername(token);
            UserDetails user = userDetailsService.loadUserByUsername(username);
            UsernamePasswordAuthenticationToken auth =
                new UsernamePasswordAuthenticationToken(user, null, user.getAuthorities());
            auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(req));
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        chain.doFilter(req, res);
    }

    private String extractBearerToken(HttpServletRequest req) {
        String header = req.getHeader(HttpHeaders.AUTHORIZATION);
        return (header != null && header.startsWith("Bearer ")) ? header.substring(7) : null;
    }
}
```

### Password Encoding — always BCrypt

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12); // cost factor ≥ 12 in prod
}
```

---

## 2. OAuth2 & Token Management

### JWT Token Service — sign with RS256, never HS256 in prod

```java
@Service
public class JwtTokenService {

    @Value("${security.jwt.private-key}")
    private RSAPrivateKey privateKey;

    @Value("${security.jwt.public-key}")
    private RSAPublicKey publicKey;

    @Value("${security.jwt.expiration-ms:3600000}")
    private long expirationMs;

    public String generateToken(UserDetails user) {
        return Jwts.builder()
            .subject(user.getUsername())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + expirationMs))
            .claim("roles", user.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority).toList())
            .signWith(privateKey, Jwts.SIG.RS256)
            .compact();
    }

    public boolean isValid(String token) {
        try {
            Jwts.parser().verifyWith(publicKey).build().parseSignedClaims(token);
            return true;
        } catch (JwtException e) {
            return false;
        }
    }

    public String extractUsername(String token) {
        return Jwts.parser().verifyWith(publicKey).build()
            .parseSignedClaims(token).getPayload().getSubject();
    }
}
```

### Refresh Token — separate, rotated, stored hashed

- Store refresh tokens as `SHA-256(token)` in DB — never raw.
- Rotate on every use (one-time use tokens).
- Expire in 7–30 days; short-lived access tokens (15 min).

---

## 3. Factory Pattern — Mandatory Construction Strategy

Never instantiate business objects with `new` in service/controller logic. Use factories.

### Abstract Factory for Security Handlers

```java
// Contract
public interface SecurityHandlerFactory {
    AuthenticationHandler create(AuthMethod method);
}

// Enum-driven dispatch factory
@Component
@RequiredArgsConstructor
public class DefaultSecurityHandlerFactory implements SecurityHandlerFactory {

    private final Map<AuthMethod, AuthenticationHandler> handlers;

    @Override
    public AuthenticationHandler create(AuthMethod method) {
        AuthenticationHandler handler = handlers.get(method);
        if (handler == null) {
            throw new UnsupportedAuthMethodException("No handler registered for: " + method);
        }
        return handler;
    }
}

// Spring auto-wires all AuthenticationHandler beans into the map via @Qualifier or a Map<String, T> injection
```

### Factory Method for Token Creation

```java
public abstract class TokenFactory {
    public abstract Token create(TokenRequest request);

    public static TokenFactory forType(TokenType type) {
        return switch (type) {
            case ACCESS  -> new AccessTokenFactory();
            case REFRESH -> new RefreshTokenFactory();
            case API_KEY -> new ApiKeyTokenFactory();
        };
    }
}
```

### Builder + Factory for complex domain objects

```java
public class AuditEventFactory {
    public static AuditEvent securityViolation(String userId, String action, String ip) {
        return AuditEvent.builder()
            .type(AuditType.SECURITY_VIOLATION)
            .userId(userId)
            .action(action)
            .sourceIp(ip)
            .timestamp(Instant.now())
            .severity(Severity.HIGH)
            .build();
    }
}
```

**Rule:** Every factory must be in its own class. Never mix factory logic into services.

---

## 4. Modular Architecture

### Module Boundaries — one module, one contract

```
com.company.app/
├── security/
│   ├── auth/          # Authentication module — JWT, OAuth2
│   ├── authz/         # Authorization module — RBAC, ACL
│   ├── audit/         # Audit logging module
│   └── crypto/        # Encryption/hashing utilities
├── user/
│   ├── api/           # Controller layer — HTTP contracts only
│   ├── domain/        # Business logic, no framework deps
│   ├── infra/         # Repository implementations, DB
│   └── factory/       # All object creation lives here
└── shared/
    ├── exception/     # Typed exceptions per domain
    ├── validation/    # Input validators
    └── event/         # Domain events
```

### Anti-patterns to reject immediately

| Anti-pattern | Why it's rejected | Fix |
|---|---|---|
| `@Service` with 500+ lines | Violates SRP | Split into domain service + infrastructure |
| `new SomeService()` in controller | Bypasses DI, untestable | Inject via constructor |
| Shared mutable state in beans | Thread safety violation | Use `@Scope("prototype")` or stateless design |
| Business logic in `@Controller` | Tight coupling | Move to `@Service` or domain object |
| `@Autowired` on field | Hidden dependency | Use constructor injection only |

### Constructor injection — always

```java
@Service
public class UserService {
    private final UserRepository userRepository;
    private final UserFactory userFactory;
    private final AuditService auditService;

    // Lombok @RequiredArgsConstructor or explicit constructor — never @Autowired on fields
    public UserService(UserRepository userRepository, UserFactory userFactory,
                       AuditService auditService) {
        this.userRepository = Objects.requireNonNull(userRepository);
        this.userFactory    = Objects.requireNonNull(userFactory);
        this.auditService   = Objects.requireNonNull(auditService);
    }
}
```

---

## 5. Input Validation & OWASP Top 10 Defense

### Bean Validation — enforce at API boundary

```java
public record CreateUserRequest(
    @NotBlank @Size(min = 3, max = 50) String username,
    @Email @NotBlank String email,
    @NotBlank @Pattern(
        regexp = "^(?=.*[A-Z])(?=.*[0-9])(?=.*[!@#$%]).{12,}$",
        message = "Password must be ≥12 chars with uppercase, digit, and special char"
    ) String password
) {}

@RestController
@RequestMapping("/api/users")
public class UserController {
    @PostMapping
    public ResponseEntity<UserResponse> create(@Valid @RequestBody CreateUserRequest req) {
        // safe — validated at boundary
    }
}
```

### SQL Injection — parameterized queries only

```java
// NEVER:  "SELECT * FROM users WHERE id = " + userId
// ALWAYS:
@Query("SELECT u FROM User u WHERE u.id = :id")
Optional<User> findById(@Param("id") Long id);
```

### XSS Prevention

- Return `Content-Type: application/json` — never raw HTML from APIs.
- Sanitize with OWASP Java HTML Sanitizer if HTML output is unavoidable.
- Set headers: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`.

### CORS — explicit allowlist, never wildcard in prod

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.company.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setAllowCredentials(true);
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}
```

### Rate Limiting — protect every public endpoint

Apply Bucket4j or Spring Cloud Gateway rate limiting on:
- `/api/auth/login` — max 5 attempts/min per IP
- `/api/auth/register` — max 3 attempts/min per IP
- `/api/public/**` — max 100 req/min per IP

---

## 6. System Design Patterns

### RBAC (Role-Based Access Control)

```java
// Method-level security
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
public UserProfile getProfile(Long userId) { ... }

// Custom permission evaluator for complex rules
@PreAuthorize("@permissionEvaluator.canAccess(authentication, #resourceId, 'READ')")
public Resource getResource(String resourceId) { ... }
```

### Audit Logging — every sensitive operation

```java
@Aspect
@Component
@RequiredArgsConstructor
public class SecurityAuditAspect {

    private final AuditService auditService;

    @AfterReturning("@annotation(audited)")
    public void logAuditedOperation(JoinPoint jp, Audited audited) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        auditService.record(AuditEventFactory.create(
            auth.getName(), jp.getSignature().getName(), audited.action()
        ));
    }
}
```

### Chain of Responsibility for Request Validation

```java
public abstract class ValidationHandler {
    private ValidationHandler next;

    public ValidationHandler setNext(ValidationHandler next) {
        this.next = next;
        return next;
    }

    public abstract ValidationResult validate(SecurityContext ctx);

    protected ValidationResult passToNext(SecurityContext ctx) {
        return next != null ? next.validate(ctx) : ValidationResult.PASS;
    }
}
```

### Secure Configuration Properties

```java
@ConfigurationProperties(prefix = "security")
@Validated
public record SecurityProperties(
    @NotBlank String jwtSecret,
    @Min(900_000) long tokenExpirationMs,         // minimum 15 min
    @NotEmpty List<String> allowedOrigins,
    @Min(10) int bcryptStrength
) {}
```

Always externalize secrets — use Vault, AWS Secrets Manager, or Kubernetes Secrets. **Never commit secrets to git.**

---

## 7. Security Checklist (apply to every PR)

```
Security Review:
- [ ] No secrets or credentials in source code
- [ ] All endpoints covered by authentication rules
- [ ] Input validated at API boundary with Bean Validation
- [ ] Parameterized queries only — zero string concatenation in SQL
- [ ] CORS locked to explicit origin allowlist
- [ ] JWT uses asymmetric signing (RS256) in production
- [ ] Password stored with BCryptPasswordEncoder(≥12)
- [ ] Sensitive operations emit audit log events
- [ ] Rate limiting on public/auth endpoints
- [ ] Dependencies scanned for CVEs (OWASP Dependency-Check or Snyk)
- [ ] Factory pattern used for all domain object creation
- [ ] No business logic in controllers
- [ ] Constructor injection used throughout (no @Autowired fields)
- [ ] Error responses reveal no internal stack traces
```

---

## Additional Resources

- For system design patterns, see [PATTERNS.md](PATTERNS.md)
- For security vulnerability examples and fixes, see [VULNERABILITIES.md](VULNERABILITIES.md)
