# Security Vulnerability Reference — Common Java/Spring Pitfalls

## OWASP Top 10 — Java/Spring Specific

### A01 — Broken Access Control

❌ Vulnerable:
```java
@GetMapping("/users/{id}/data")
public UserData getData(@PathVariable Long id) {
    return userService.getData(id); // ANY authenticated user can access ANY user's data
}
```

✅ Fixed:
```java
@GetMapping("/users/{id}/data")
@PreAuthorize("#id == authentication.principal.id or hasRole('ADMIN')")
public UserData getData(@PathVariable Long id) {
    return userService.getData(id);
}
```

---

### A02 — Cryptographic Failures

❌ Vulnerable:
```java
// MD5 / SHA-1 are broken for passwords
String hash = DigestUtils.md5Hex(password);
// Symmetric JWT signing — key compromise = all tokens compromised
Jwts.builder().signWith(Keys.hmacShaKeyFor(secret.getBytes())).compact();
```

✅ Fixed:
```java
// BCrypt with cost ≥ 12
passwordEncoder.encode(password);
// Asymmetric JWT — private key signs, public key verifies
Jwts.builder().signWith(rsaPrivateKey, Jwts.SIG.RS256).compact();
```

---

### A03 — Injection

❌ Vulnerable:
```java
String query = "SELECT * FROM users WHERE email = '" + email + "'";
jdbcTemplate.queryForObject(query, User.class);
```

✅ Fixed:
```java
jdbcTemplate.queryForObject(
    "SELECT * FROM users WHERE email = ?",
    new Object[]{email},
    User.class
);
```

---

### A07 — Identification and Authentication Failures

Checklist:
- [ ] No default credentials in any configuration
- [ ] Account lockout after N failed attempts (track per username + IP)
- [ ] Password reset tokens are time-limited (≤15 min), single-use, hashed in DB
- [ ] Sessions invalidated on logout (`SecurityContextHolder.clearContext()`)
- [ ] Remember-me tokens rotated on use

---

### A09 — Security Logging and Monitoring Failures

Every security event MUST be logged with:
- `timestamp` (ISO-8601 with timezone)
- `userId` or `anonymous`
- `action` (login, logout, access-denied, password-change, etc.)
- `sourceIp`
- `outcome` (SUCCESS / FAILURE)
- `reason` (on failure)

Never log passwords, tokens, or PII in plain text.

---

## Common Spring Boot Misconfigurations

| Misconfiguration | Risk | Fix |
|---|---|---|
| `spring.security.enabled=false` | All endpoints public | Remove — never disable security |
| Actuator endpoints exposed without auth | Info disclosure | `management.endpoint.*.access=restricted` |
| `server.error.include-stacktrace=always` | Stack trace leaks in prod | Set to `never` in prod profile |
| Wildcard CORS `*` | Full CORS bypass | Explicit allowlist only |
| H2 console enabled in prod | Direct DB access | Disable for non-dev profiles |
| `ddl-auto=create-drop` in prod | Data loss | Use `validate` in prod |

---

## Dependency Vulnerability Scanning

Run OWASP Dependency-Check on every build:

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>9.0.0</version>
    <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS> <!-- fail on HIGH+ CVEs -->
    </configuration>
</plugin>
```

Or with Gradle:
```groovy
plugins { id "org.owasp.dependencycheck" version "9.0.0" }
dependencyCheck { failBuildOnCVSS = 7 }
```
