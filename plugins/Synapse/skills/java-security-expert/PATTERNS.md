# System Design Patterns Reference

## Creational Patterns

### Factory Method
Use when subclasses decide which class to instantiate.
```java
public abstract class NotificationFactory {
    public abstract Notification create(NotificationType type);
}

@Component
public class EmailNotificationFactory extends NotificationFactory {
    @Override
    public Notification create(NotificationType type) {
        return new EmailNotification(smtpConfig, type);
    }
}
```

### Abstract Factory
Use when you need families of related objects.
```java
public interface SecurityComponentFactory {
    TokenValidator createTokenValidator();
    PasswordHasher createPasswordHasher();
    AuditLogger createAuditLogger();
}

@Component
public class ProdSecurityComponentFactory implements SecurityComponentFactory {
    @Override public TokenValidator createTokenValidator() { return new JwtValidator(rsaPublicKey); }
    @Override public PasswordHasher createPasswordHasher()  { return new BCryptHasher(12); }
    @Override public AuditLogger createAuditLogger()        { return new DatabaseAuditLogger(dataSource); }
}
```

### Builder (with validation)
```java
@Builder
@Validated
public class SecurityPolicy {
    @Min(12) private final int passwordMinLength;
    @NotNull  private final TokenExpiry accessTokenExpiry;
    @NotNull  private final TokenExpiry refreshTokenExpiry;
    @NotEmpty private final Set<String> allowedOrigins;

    // Builder validates before construction
    public static class SecurityPolicyBuilder {
        public SecurityPolicy build() {
            SecurityPolicy policy = new SecurityPolicy(this);
            // cross-field validation
            if (policy.accessTokenExpiry.isLongerThan(policy.refreshTokenExpiry)) {
                throw new IllegalStateException("Access token cannot outlive refresh token");
            }
            return policy;
        }
    }
}
```

---

## Structural Patterns

### Decorator — adding security to existing services
```java
public class AuditedUserService implements UserService {
    private final UserService delegate;
    private final AuditService audit;

    @Override
    public User createUser(CreateUserRequest req) {
        User user = delegate.createUser(req);
        audit.record(AuditEventFactory.userCreated(user.getId()));
        return user;
    }
}
```

### Facade — hide complexity of security subsystems
```java
@Service
public class AuthFacade {
    private final TokenService tokenService;
    private final UserDetailsService userService;
    private final AuditService auditService;
    private final RateLimiter rateLimiter;

    public AuthResponse login(LoginRequest req) {
        rateLimiter.checkOrThrow(req.ip());
        UserDetails user = userService.loadUserByUsername(req.username());
        auditService.record(AuditEventFactory.loginAttempt(req.username(), req.ip()));
        return AuthResponse.of(tokenService.generateAccessToken(user),
                               tokenService.generateRefreshToken(user));
    }
}
```

### Proxy — security enforcement point
```java
@Component
public class SecuredRepositoryProxy implements UserRepository {
    private final UserRepository target;
    private final SecurityContext ctx;

    @Override
    public Optional<User> findById(Long id) {
        if (!ctx.hasPermission("USER_READ", id)) {
            throw new AccessDeniedException("Cannot read user " + id);
        }
        return target.findById(id);
    }
}
```

---

## Behavioral Patterns

### Strategy — pluggable authentication strategies
```java
public interface AuthenticationStrategy {
    Authentication authenticate(Credentials credentials);
    boolean supports(AuthMethod method);
}

@Component
public class JwtAuthenticationStrategy implements AuthenticationStrategy {
    @Override public boolean supports(AuthMethod m) { return m == AuthMethod.JWT; }
    @Override public Authentication authenticate(Credentials c) { /* validate JWT */ }
}

@Component
public class ApiKeyAuthenticationStrategy implements AuthenticationStrategy {
    @Override public boolean supports(AuthMethod m) { return m == AuthMethod.API_KEY; }
    @Override public Authentication authenticate(Credentials c) { /* validate API key */ }
}

// Aggregator — picks correct strategy
@Service
@RequiredArgsConstructor
public class AuthenticationDispatcher {
    private final List<AuthenticationStrategy> strategies;

    public Authentication authenticate(AuthMethod method, Credentials credentials) {
        return strategies.stream()
            .filter(s -> s.supports(method))
            .findFirst()
            .orElseThrow(() -> new UnsupportedAuthMethodException(method.name()))
            .authenticate(credentials);
    }
}
```

### Observer — security events
```java
// Publish domain events — don't couple security logic to side effects
@Service
public class LoginService {
    private final ApplicationEventPublisher publisher;

    public void login(LoginRequest req) {
        // ... authenticate ...
        publisher.publishEvent(new LoginSuccessEvent(user, req.ip(), Instant.now()));
    }
}

@Component
public class BruteForceDetector implements ApplicationListener<LoginFailureEvent> {
    @Override
    public void onApplicationEvent(LoginFailureEvent event) {
        // track failures per IP, lock accounts as needed
    }
}
```

### Chain of Responsibility — layered validation pipeline
```java
// Arrange validators as a pipeline
ValidationHandler pipeline = new TokenExpiryValidator()
    .setNext(new TokenSignatureValidator()
    .setNext(new UserActiveValidator()
    .setNext(new IpAllowlistValidator())));

ValidationResult result = pipeline.validate(context);
```

### Template Method — standardized security operation flow
```java
public abstract class SecureOperation<Request, Response> {
    // Fixed algorithm — subclasses fill in specifics
    public final Response execute(Request req) {
        validateInput(req);                 // Step 1 — mandatory, always
        checkAuthorization(req);            // Step 2 — mandatory, always
        Response result = doExecute(req);   // Step 3 — subclass implements
        auditLog(req, result);              // Step 4 — mandatory, always
        return result;
    }

    protected abstract void validateInput(Request req);
    protected abstract void checkAuthorization(Request req);
    protected abstract Response doExecute(Request req);
    protected abstract void auditLog(Request req, Response result);
}
```

---

## Microservice Security Patterns

### Service-to-Service Authentication (mTLS or service account JWT)
- Each service has its own service account token.
- Validate `iss` and `aud` claims on inter-service calls.
- Use mTLS in zero-trust networks.

### API Gateway as Security Perimeter
- Authentication/rate-limiting happens at gateway — services trust the gateway.
- Pass user context as verified JWT claim, not re-authenticate in each service.

### Secret Rotation without Downtime
- Store secrets in Vault with dynamic secret lease.
- Use dual-active credentials during rotation window.
- Never restart the entire service on secret rotation — reload via `@RefreshScope`.
