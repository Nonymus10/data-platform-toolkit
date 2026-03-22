# OWASP Security Enforcement Reference

This document provides concrete code patterns for enforcing OWASP Top 10 protections across common backend frameworks. The service-surgeon agent MUST reference these patterns when making any code changes.

---

## A01: Broken Access Control

### Spring Boot / Spring Security
```java
// Every endpoint must have explicit authorization — never rely on "authenticated by default"
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(auth -> auth
        .requestMatchers("/api/public/**").permitAll()
        .requestMatchers("/api/admin/**").hasRole("ADMIN")
        .requestMatchers(HttpMethod.DELETE, "/api/**").hasRole("ADMIN")
        .anyRequest().authenticated()  // ← deny by default
    );
    return http.build();
}

// Method-level: always check ownership
@PreAuthorize("hasRole('ADMIN') or @ownershipCheck.isOwner(authentication, #id)")
@GetMapping("/api/v1/users/{id}")
public UserResponse getUser(@PathVariable Long id) { ... }
```

### FastAPI / Python
```python
# Dependency injection for auth — applied per route
async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    user = await verify_token(token)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid credentials")
    return user

async def require_admin(user: User = Depends(get_current_user)) -> User:
    if user.role != Role.ADMIN:
        raise HTTPException(status_code=403, detail="Admin access required")
    return user

@router.get("/api/v1/users/{user_id}")
async def get_user(user_id: int, current_user: User = Depends(get_current_user)):
    if current_user.id != user_id and current_user.role != Role.ADMIN:
        raise HTTPException(status_code=403, detail="Access denied")
    return await user_service.get(user_id)
```

### Express / Node.js
```typescript
// Middleware chain for auth
const authenticate = (req: Request, res: Response, next: NextFunction) => {
    const token = req.headers.authorization?.replace('Bearer ', '');
    if (!token) return res.status(401).json({ error: 'No token provided' });
    try {
        req.user = verifyToken(token);
        next();
    } catch {
        res.status(401).json({ error: 'Invalid token' });
    }
};

const authorize = (...roles: string[]) => (req: Request, res: Response, next: NextFunction) => {
    if (!roles.includes(req.user.role)) {
        return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
};

router.delete('/api/v1/users/:id', authenticate, authorize('ADMIN'), deleteUser);
```

### Common Violations to Catch
| Violation | Detection Pattern | Fix |
|-----------|------------------|-----|
| Endpoint with no auth check | Route handler without auth middleware/annotation | Add auth explicitly |
| IDOR (Insecure Direct Object Reference) | `findById(id)` without ownership check | Add ownership verification |
| Privilege escalation | User can modify their own role | Server-side role enforcement, ignore role in user-submitted payloads |
| Horizontal access | User A can access User B's resources | Check `resource.ownerId == currentUser.id` |
| Missing function-level access | Admin endpoints accessible to regular users | Explicit role checks per endpoint |

---

## A02: Cryptographic Failures

### Password Storage
```java
// ALWAYS BCrypt with cost factor ≥ 12
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);
}
```

### Token Signing
```java
// ALWAYS RS256 (asymmetric) in production — HS256 only for local dev
Jwts.builder()
    .signWith(rsaPrivateKey, Jwts.SIG.RS256)
    .compact();
```

### Data at Rest
```java
// Encrypt sensitive fields before persistence
@Convert(converter = EncryptedStringConverter.class)
@Column(name = "ssn")
private String socialSecurityNumber;
```

### Common Violations
| Violation | Fix |
|-----------|-----|
| MD5/SHA1 for passwords | BCrypt(12+) or Argon2 |
| HS256 with hardcoded secret | RS256 with key pair, secret in Vault |
| Sensitive data in plain text DB columns | Application-level encryption or DB-level TDE |
| Secrets in application.yml committed to git | Environment variables or secret manager |
| HTTP for internal service communication | mTLS or at minimum HTTPS |

---

## A03: Injection

### SQL Injection Prevention
```java
// ALWAYS parameterized — Spring Data JPA
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);

// NEVER string concatenation
// REJECT: "SELECT * FROM users WHERE email = '" + email + "'"
```

```python
# SQLAlchemy — always use bound parameters
result = db.execute(
    text("SELECT * FROM users WHERE email = :email"),
    {"email": email}
)
```

```typescript
// Prisma — parameterized by default
const user = await prisma.user.findUnique({ where: { email } });

// Raw query — use $queryRaw with template literals (auto-parameterized)
const users = await prisma.$queryRaw`SELECT * FROM users WHERE email = ${email}`;
```

### Command Injection Prevention
```java
// NEVER pass user input to Runtime.exec or ProcessBuilder without validation
// If shell commands are absolutely necessary, use allowlists
List<String> allowedCommands = List.of("ls", "cat", "head");
if (!allowedCommands.contains(userCommand)) {
    throw new SecurityException("Command not allowed");
}
```

### LDAP Injection Prevention
```java
// Escape special characters in LDAP queries
String safeName = name.replace("\\", "\\\\")
    .replace("*", "\\*").replace("(", "\\(").replace(")", "\\)");
```

---

## A04: Insecure Design

### Threat Modeling Checklist for New Features
```
For every new feature, answer:
1. Who should be able to use this? (auth + authz)
2. What's the worst thing a malicious user could do with this? (abuse cases)
3. What happens if this is called 10,000 times per second? (rate limiting)
4. What sensitive data does this handle? (encryption, logging exclusion)
5. What happens if the downstream service is down? (resilience)
6. Can this be used to exfiltrate data? (output validation)
7. Does this create a new attack surface? (CORS, public endpoints)
```

### Rate Limiting
```java
// Auth endpoints — strict limits
@Bean
public RateLimiter loginRateLimiter() {
    return RateLimiter.of("login", RateLimiterConfig.custom()
        .limitRefreshPeriod(Duration.ofMinutes(1))
        .limitForPeriod(5)           // 5 attempts per minute
        .timeoutDuration(Duration.ZERO) // fail immediately
        .build());
}
```

### Business Logic Limits
```java
// Server-side enforcement — never trust client
public Order createOrder(OrderRequest req) {
    if (req.getQuantity() > MAX_ORDER_QUANTITY) {
        throw new BusinessRuleException("Order quantity exceeds maximum");
    }
    if (req.getDiscount() > 0 && !currentUser.hasRole("ADMIN")) {
        throw new AccessDeniedException("Only admins can apply discounts");
    }
}
```

---

## A05: Security Misconfiguration

### CORS — Explicit Allowlist
```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of(
        "https://app.company.com",
        "https://admin.company.com"
    ));  // NEVER use "*" with credentials
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}
```

### Security Headers
```java
// Add to every response
http.headers(h -> h
    .contentTypeOptions(Customizer.withDefaults())     // X-Content-Type-Options: nosniff
    .frameOptions(f -> f.deny())                        // X-Frame-Options: DENY
    .httpStrictTransportSecurity(hsts -> hsts
        .maxAgeInSeconds(31536000).includeSubDomains(true))
    .referrerPolicy(r -> r.policy(ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN))
);
```

### Debug/Dev Features
```yaml
# application-prod.yml — explicitly disable debug features
spring:
  jpa:
    show-sql: false
  devtools:
    restart:
      enabled: false
management:
  endpoints:
    web:
      exposure:
        include: health,prometheus  # NOT *
```

---

## A06: Vulnerable and Outdated Components

### Before Adding Any Dependency
```
1. Check for known CVEs: https://nvd.nist.gov/ or `npm audit` / `mvn dependency-check:check`
2. Verify the package is actively maintained (last commit < 6 months)
3. Pin exact version (never use `latest` or `*`)
4. Prefer well-known libraries over obscure alternatives
5. Check license compatibility
```

---

## A07: Identification and Authentication Failures

### Brute Force Protection
```java
// Track failed attempts per IP and per username
@Component
public class LoginAttemptService {
    private final LoadingCache<String, Integer> attemptsCache;

    public void loginFailed(String key) {
        int attempts = attemptsCache.get(key) + 1;
        attemptsCache.put(key, attempts);
    }

    public boolean isBlocked(String key) {
        return attemptsCache.get(key) >= MAX_ATTEMPTS; // e.g., 5
    }
}
```

### Token Best Practices
| Token Type | Lifetime | Storage | Rotation |
|-----------|----------|---------|----------|
| Access token (JWT) | 15 minutes | Memory only (never localStorage) | On expiry via refresh token |
| Refresh token | 7–30 days | HttpOnly Secure cookie or DB (hashed) | Rotate on every use (one-time) |
| API key | Long-lived | Server-side only, hashed in DB | Manual rotation with overlap period |

---

## A08: Software and Data Integrity Failures

### Deserialization Safety
```java
// NEVER deserialize untrusted data with Java native serialization
// Use JSON with explicit type mapping
ObjectMapper mapper = new ObjectMapper();
mapper.activateDefaultTyping(
    BasicPolymorphicTypeValidator.builder()
        .allowIfSubType("com.company.dto.")  // allowlist, not denylist
        .build(),
    ObjectMapper.DefaultTyping.NON_FINAL
);
```

### Webhook Signature Verification
```java
// ALWAYS verify webhook signatures before processing
public boolean verifyWebhookSignature(String payload, String signature, String secret) {
    String computed = HmacUtils.hmacSha256Hex(secret, payload);
    return MessageDigest.isEqual(computed.getBytes(), signature.getBytes());
}
```

---

## A09: Security Logging and Monitoring Failures

### What MUST Be Logged
```java
// Auth events — always log
log.info("login_success user={} ip={}", username, clientIp);
log.warn("login_failure user={} ip={} reason={}", username, clientIp, reason);
log.warn("access_denied user={} resource={} action={}", username, resource, action);
log.warn("input_validation_failure endpoint={} field={} value_type={}", endpoint, field, valueType);

// NEVER log
// log.info("Password: {}", password);         ← NEVER
// log.info("Token: {}", jwtToken);            ← NEVER
// log.info("SSN: {}", user.getSsn());         ← NEVER
// log.info("Credit card: {}", cardNumber);    ← NEVER
```

### Structured Logging Format
```java
// Use MDC for correlation
MDC.put("correlationId", requestId);
MDC.put("userId", currentUser.getId());
MDC.put("endpoint", request.getRequestURI());
log.info("order_created orderId={} amount={}", order.getId(), order.getAmount());
```

---

## A10: Server-Side Request Forgery (SSRF)

### URL Validation
```java
// ALWAYS validate URLs before making server-side requests
public URI validateUrl(String url) {
    URI uri = URI.create(url);

    // Block internal networks
    InetAddress address = InetAddress.getByName(uri.getHost());
    if (address.isLoopbackAddress() || address.isSiteLocalAddress() || address.isLinkLocalAddress()) {
        throw new SecurityException("Internal network access denied");
    }

    // Allowlist schemes
    if (!List.of("http", "https").contains(uri.getScheme())) {
        throw new SecurityException("Unsupported URL scheme");
    }

    // Allowlist domains if possible
    if (!ALLOWED_DOMAINS.contains(uri.getHost())) {
        throw new SecurityException("Domain not in allowlist");
    }

    return uri;
}
```
