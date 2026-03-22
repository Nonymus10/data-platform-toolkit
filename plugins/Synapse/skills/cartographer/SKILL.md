---
name: cartographer
description: Maps the complete flow of a service API by API — traces every endpoint from route to handler to service to repository to database, identifies middleware chains, custom library calls, inter-service communication, unused code, and produces a structured, exhaustive documentation artifact that downstream agents can consume to modify, refactor, test, or migrate the codebase.
---

# Service Cartographer Skill

Produces a complete, structured map of a service's internals — every API endpoint, every function in the call chain, every file involved, every external dependency touched. The output is designed to be consumed by other agents as a knowledge base.

## Core Philosophy

- **Trace, don't summarize.** Every claim must reference a concrete file path and line number.
- **Follow the code, not the docs.** Documentation lies. Code is the source of truth.
- **Cross every boundary.** If a function calls into a custom in-repo library, trace into that library. If it makes an HTTP call to another internal service, document the contract.
- **Structure for machines.** The output must be parseable by another agent — consistent headings, consistent field names, no free-form prose in structured sections.

---

## Phase 1: Project Reconnaissance

Before tracing any API, build a complete picture of the project. This section is generated **once per service**.

### 1.1 Technology Stack Detection

Scan build files, package manifests, and configuration to auto-detect the stack:

```
## Technology Stack

### Language & Runtime
- **Language:** [e.g., Java 17, Python 3.11, TypeScript 5.3, Go 1.22]
- **Runtime:** [e.g., JVM 17 (GraalVM), Node.js 20 LTS, CPython 3.11]
- **Package Manager:** [e.g., Maven 3.9, Gradle 8.5, Poetry 1.7, pnpm 8.x]

### Framework & Server
- **Web Framework:** [e.g., Spring Boot 3.2, FastAPI 0.115, Express 4.18, Gin 1.9]
- **Server:** [e.g., Embedded Tomcat, Uvicorn, Node HTTP, net/http]
- **API Style:** [REST, GraphQL, gRPC, mixed]
- **API Documentation:** [OpenAPI/Swagger, GraphQL schema, protobuf definitions]

### Data Layer
- **ORM/Data Access:** [e.g., JPA/Hibernate, SQLAlchemy, Prisma, GORM]
- **Database(s):** [e.g., PostgreSQL 15, Redis 7, MongoDB 7, Elasticsearch 8]
- **Migration Tool:** [e.g., Flyway, Alembic, Prisma Migrate, golang-migrate]
- **Caching:** [e.g., Redis, Caffeine, node-cache, in-memory]

### Authentication & Security
- **Auth Framework:** [e.g., Spring Security, Passport.js, python-jose, custom middleware]
- **Auth Method:** [JWT, OAuth2, session-based, API keys, mTLS]
- **Authorization Model:** [RBAC, ABAC, ACL, custom]

### Messaging & Async
- **Message Broker:** [e.g., Kafka, RabbitMQ, SQS, Redis Pub/Sub, none]
- **Async Patterns:** [event-driven, CQRS, saga, none]
- **Background Jobs:** [e.g., Celery, Spring Batch, BullMQ, none]

### Observability
- **Logging:** [e.g., SLF4J/Logback, structlog, Winston, zerolog]
- **Metrics:** [e.g., Micrometer/Prometheus, StatsD, OpenTelemetry]
- **Tracing:** [e.g., OpenTelemetry, Jaeger, X-Ray, none]
- **Health Checks:** [e.g., /actuator/health, /healthz, custom]

### Build & Quality
- **Build Tool:** [e.g., Maven, Gradle, Webpack, esbuild, go build]
- **Linter:** [e.g., Checkstyle, Ruff, ESLint, golangci-lint]
- **Formatter:** [e.g., google-java-format, Black, Prettier, gofmt]
- **Type Checker:** [e.g., compiler-enforced, mypy, TypeScript strict, none]
- **Testing Framework:** [e.g., JUnit 5, pytest, Jest/Vitest, go test]
- **CI/CD:** [e.g., GitHub Actions, GitLab CI, Jenkins, CircleCI]

### Deployment
- **Containerization:** [e.g., Docker, Buildpacks, none]
- **Orchestration:** [e.g., Kubernetes, ECS, Lambda, bare metal]
- **Infrastructure as Code:** [e.g., Terraform, Pulumi, CloudFormation, none]
```

### 1.2 Complete File Inventory

Generate a full file tree with annotations:

```
## File Inventory

### Structure
[PROJECT-ROOT]/
├── src/main/java/com/example/service/
│   ├── Application.java                    # [ENTRYPOINT] Spring Boot main class
│   ├── config/
│   │   ├── SecurityConfig.java             # [CONFIG] Spring Security filter chain
│   │   ├── CorsConfig.java                 # [CONFIG] CORS configuration
│   │   ├── RedisConfig.java                # [CONFIG] Redis connection factory
│   │   └── SwaggerConfig.java              # [CONFIG] OpenAPI documentation
│   ├── controller/
│   │   ├── UserController.java             # [API] /api/v1/users — CRUD operations
│   │   ├── AuthController.java             # [API] /api/v1/auth — login, register, refresh
│   │   └── OrderController.java            # [API] /api/v1/orders — order lifecycle
│   ├── service/
│   │   ├── UserService.java                # [SERVICE] User business logic
│   │   ├── AuthService.java                # [SERVICE] Authentication orchestration
│   │   └── OrderService.java               # [SERVICE] Order processing
│   ├── repository/
│   │   ├── UserRepository.java             # [REPO] JPA repository for users table
│   │   └── OrderRepository.java            # [REPO] JPA repository for orders table
│   ├── model/
│   │   ├── entity/
│   │   │   ├── User.java                   # [ENTITY] users table mapping
│   │   │   └── Order.java                  # [ENTITY] orders table mapping
│   │   ├── dto/
│   │   │   ├── UserRequest.java            # [DTO-IN] Create/update user payload
│   │   │   ├── UserResponse.java           # [DTO-OUT] User API response
│   │   │   └── OrderRequest.java           # [DTO-IN] Create order payload
│   │   └── enums/
│   │       ├── UserRole.java               # [ENUM] ADMIN, USER, VIEWER
│   │       └── OrderStatus.java            # [ENUM] PENDING, CONFIRMED, SHIPPED, DELIVERED
│   ├── middleware/
│   │   ├── JwtAuthFilter.java              # [FILTER] JWT token validation
│   │   ├── RateLimitFilter.java            # [FILTER] Request rate limiting
│   │   └── RequestLoggingFilter.java       # [FILTER] HTTP request/response logging
│   ├── exception/
│   │   ├── GlobalExceptionHandler.java     # [ERROR] @ControllerAdvice error mapping
│   │   ├── ResourceNotFoundException.java  # [ERROR] 404 exception
│   │   └── UnauthorizedException.java      # [ERROR] 401 exception
│   ├── util/
│   │   ├── JwtUtil.java                    # [UTIL] Token generation and parsing
│   │   └── DateUtil.java                   # [UTIL] Date formatting helpers
│   └── client/
│       ├── PaymentClient.java              # [CLIENT] External payment service HTTP client
│       └── NotificationClient.java         # [CLIENT] Internal notification service client
├── src/main/resources/
│   ├── application.yml                     # [CONFIG] Main configuration
│   ├── application-dev.yml                 # [CONFIG] Dev profile overrides
│   └── db/migration/
│       ├── V1__create_users.sql            # [MIGRATION] Users table DDL
│       └── V2__create_orders.sql           # [MIGRATION] Orders table DDL
└── src/test/java/...                       # [TEST] Mirror of main structure

### File Classification Summary
| Category     | Count | Files |
|-------------|-------|-------|
| API Controllers | N | list... |
| Services    | N     | list... |
| Repositories | N    | list... |
| Entities    | N     | list... |
| DTOs        | N     | list... |
| Filters/Middleware | N | list... |
| Configuration | N   | list... |
| Utilities   | N     | list... |
| Clients (external) | N | list... |
| Clients (internal) | N | list... |
| Migrations  | N     | list... |
| Tests       | N     | list... |
| Unused/Orphaned | N | list... |
```

### 1.3 Entrypoint & Bootstrap Analysis

Document how the application starts:

```
## Application Bootstrap

### Entrypoint
- **File:** [path]:[line]
- **Main Class/Function:** [name]
- **Bootstrap Sequence:**
  1. [e.g., Spring context initializes → component scan on com.example.service]
  2. [e.g., Auto-configuration loads SecurityConfig, CorsConfig, RedisConfig]
  3. [e.g., Flyway runs pending migrations]
  4. [e.g., Embedded Tomcat starts on port 8080]
  5. [e.g., Health check endpoint becomes available at /actuator/health]

### Configuration Loading Order
1. [e.g., application.yml → base config]
2. [e.g., application-{profile}.yml → profile overrides]
3. [e.g., Environment variables → final overrides]
4. [e.g., Vault/Secrets Manager → sensitive values injected at runtime]

### Environment Variables & Configuration Surface
| Variable | Source | Used By | Required | Default |
|----------|--------|---------|----------|---------|
| DB_URL   | env    | DataSourceConfig.java:15 | yes | none |
| JWT_SECRET | vault | JwtUtil.java:22 | yes | none |
| REDIS_HOST | env  | RedisConfig.java:10 | no | localhost |
```

---

## Phase 2: API-by-API Flow Mapping

For **every** API endpoint discovered, produce this exact structure. Do not skip any endpoint.

### 2.1 Per-API Documentation Block

```
---

## API: [METHOD] [PATH]

### Summary
| Field | Value |
|-------|-------|
| **HTTP Method** | GET / POST / PUT / PATCH / DELETE |
| **Path** | /api/v1/resource/{id} |
| **Auth Required** | Yes / No |
| **Required Roles** | [ADMIN, USER] or "any authenticated" or "public" |
| **Rate Limited** | Yes (100 req/min per IP) / No |
| **Request Content-Type** | application/json / multipart/form-data / none |
| **Response Content-Type** | application/json |
| **Success Status** | 200 / 201 / 204 |
| **Error Statuses** | 400, 401, 403, 404, 409, 500 |

### Request Schema
| Field | Type | Required | Validation | Notes |
|-------|------|----------|------------|-------|
| name  | string | yes | @NotBlank, @Size(3,50) | — |
| email | string | yes | @Email | unique constraint in DB |
| role  | enum | no | UserRole enum | defaults to USER |

### Response Schema
| Field | Type | Nullable | Notes |
|-------|------|----------|-------|
| id    | long | no | auto-generated |
| name  | string | no | — |
| email | string | no | masked in non-admin responses |
| createdAt | ISO-8601 | no | — |

### Path Parameters
| Param | Type | Validation | Notes |
|-------|------|------------|-------|
| id    | long | @Min(1)    | — |

### Query Parameters
| Param | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| page  | int  | no       | 0       | zero-indexed |
| size  | int  | no       | 20      | max 100 |

### Middleware / Filter Chain (in execution order)
| Order | Filter/Middleware | File | Line | Purpose |
|-------|------------------|------|------|---------|
| 1 | RequestLoggingFilter | middleware/RequestLoggingFilter.java | 18 | Logs method, URI, duration |
| 2 | RateLimitFilter | middleware/RateLimitFilter.java | 25 | 100 req/min per IP |
| 3 | JwtAuthFilter | middleware/JwtAuthFilter.java | 30 | Extracts and validates Bearer token |
| 4 | Spring Security FilterChain | config/SecurityConfig.java | 42 | Role-based access check |

### Complete Call Chain
Trace every function call from the HTTP handler to the deepest layer:

| Step | Layer | File | Line | Function Signature | Purpose |
|------|-------|------|------|--------------------|---------|
| 1 | Controller | controller/UserController.java | 45 | `createUser(@Valid @RequestBody CreateUserRequest req): ResponseEntity<UserResponse>` | HTTP handler — validates request body |
| 2 | Service | service/UserService.java | 78 | `createUser(CreateUserRequest req): User` | Checks for duplicate email, hashes password |
| 3 | Repository | repository/UserRepository.java | 12 | `existsByEmail(String email): boolean` | Duplicate check query |
| 4 | Util | util/PasswordUtil.java | 30 | `hash(String raw): String` | BCrypt hashing |
| 5 | Repository | repository/UserRepository.java | 8 | `save(User entity): User` | JPA persist |
| 6 | Client | client/NotificationClient.java | 55 | `sendWelcomeEmail(String email, String name): void` | HTTP POST to notification-service |
| 7 | Service | service/UserService.java | 95 | `return UserMapper.toResponse(savedUser)` | Entity → DTO mapping |

### Custom Library Calls
If any step in the call chain invokes code from a custom in-repo library (not a third-party dependency), trace into it:

| Library | Module/Package | File | Line | Function | Purpose |
|---------|---------------|------|------|----------|---------|
| common-auth | com.example.common.auth | libs/common-auth/src/.../TokenValidator.java | 40 | `validate(String token): Claims` | Shared JWT validation logic used by multiple services |
| common-models | com.example.common.models | libs/common-models/src/.../AuditEvent.java | 15 | `AuditEvent.builder()...build()` | Shared audit event model |

### Database Operations
| Operation | Table | Type | File | Line | Query/Method |
|-----------|-------|------|------|------|-------------|
| Read | users | SELECT | UserRepository.java | 12 | `existsByEmail(email)` — checks unique constraint |
| Write | users | INSERT | UserRepository.java | 8 | `save(entity)` — JPA managed insert |
| Read | audit_log | INSERT | AuditRepository.java | 20 | `save(auditEvent)` — async via @Async |

### External Service Calls
| Service | Protocol | File | Line | Method | Endpoint/Topic | Timeout | Retry | Circuit Breaker |
|---------|----------|------|------|--------|----------------|---------|-------|-----------------|
| notification-service | HTTP POST | NotificationClient.java | 55 | `sendWelcomeEmail()` | POST /api/v1/notifications/email | 5s | 3x exponential | yes (Resilience4j) |
| payment-service | gRPC | PaymentClient.java | 30 | `charge()` | PaymentService/Charge | 10s | 2x | yes |
| order-events | Kafka | OrderProducer.java | 22 | `publishOrderCreated()` | topic: order.created | — | built-in | — |

### Inter-Service Communication Map
| Direction | Service | Protocol | Contract File | Purpose |
|-----------|---------|----------|--------------|---------|
| OUTBOUND | notification-service | REST | client/NotificationClient.java | Welcome emails, password reset |
| OUTBOUND | payment-service | gRPC | client/PaymentClient.java | Payment processing |
| INBOUND | api-gateway | REST | — | Receives proxied user requests |
| PUBLISH | Kafka: order.created | async | producer/OrderProducer.java | Order lifecycle events |
| SUBSCRIBE | Kafka: payment.completed | async | consumer/PaymentConsumer.java | Payment confirmation |

### Error Handling Path
| Error Condition | Exception Thrown | File | Line | HTTP Status | Response Body |
|----------------|-----------------|------|------|-------------|---------------|
| Duplicate email | DuplicateResourceException | service/UserService.java | 82 | 409 | `{"error": "Email already registered"}` |
| User not found | ResourceNotFoundException | service/UserService.java | 60 | 404 | `{"error": "User not found"}` |
| Invalid JWT | UnauthorizedException | middleware/JwtAuthFilter.java | 45 | 401 | `{"error": "Invalid or expired token"}` |
| Validation failure | MethodArgumentNotValidException | (framework) | — | 400 | `{"errors": [{field, message}]}` |
| Notification service down | (caught, logged) | client/NotificationClient.java | 62 | — | Does not fail the API — fire-and-forget |
| Unhandled exception | Exception | exception/GlobalExceptionHandler.java | 30 | 500 | `{"error": "Internal server error"}` |

### Data Transformation Pipeline
Track how data changes shape as it flows through layers:

| Step | Layer | Input Type | Output Type | Transformation | File:Line |
|------|-------|-----------|-------------|----------------|-----------|
| 1 | Controller | HTTP JSON body | CreateUserRequest (DTO) | Jackson deserialization + Bean Validation | UserController.java:45 |
| 2 | Service | CreateUserRequest | User (entity) | Manual mapping, password hashing, default role | UserService.java:80 |
| 3 | Repository | User (entity) | User (entity) | JPA managed, ID generated | UserRepository.java:8 |
| 4 | Service | User (entity) | UserResponse (DTO) | UserMapper.toResponse — strips password, adds computed fields | UserMapper.java:15 |
| 5 | Controller | UserResponse (DTO) | HTTP JSON response | Jackson serialization | UserController.java:48 |

---
```

Repeat this exact block for **every** endpoint. No endpoint is skipped.

---

## Phase 3: Cross-Cutting Analysis

After all APIs are mapped, produce these service-wide sections.

### 3.1 Module Dependency Graph

```
## Module Dependency Graph

### Internal Dependencies (imports between packages)
| Source Module | Depends On | Type | Notes |
|--------------|-----------|------|-------|
| controller | service | direct | All controllers inject services |
| service | repository | direct | Data access |
| service | client | direct | External calls |
| service | util | direct | Helpers |
| middleware | util | direct | JwtUtil used in JwtAuthFilter |
| config | middleware | direct | Registers filters |

### Circular Dependencies
| Module A | Module B | Via | Severity |
|----------|----------|-----|----------|
| (none found) | — | — | — |

### Custom In-Repo Library Dependencies
| Library | Location | Used By | Purpose |
|---------|----------|---------|---------|
| common-auth | libs/common-auth/ | middleware/JwtAuthFilter, service/AuthService | Shared JWT validation and token models |
| common-models | libs/common-models/ | service/*, client/* | Shared DTOs and event models |
| common-utils | libs/common-utils/ | util/* | String, date, and encryption helpers |

### Library Internal Structure
For each custom library, provide a mini file-tree and public API surface:

**common-auth** (`libs/common-auth/`)
| File | Public Functions/Classes | Used By (in main service) |
|------|------------------------|--------------------------|
| TokenValidator.java | `validate(String): Claims`, `isExpired(String): boolean` | JwtAuthFilter.java:35, AuthService.java:50 |
| TokenGenerator.java | `generate(UserDetails): String` | AuthService.java:72 |
| AuthConstants.java | `HEADER_NAME`, `TOKEN_PREFIX`, `CLAIM_ROLES` | JwtAuthFilter.java:20 |
```

### 3.2 Unused Code Report

```
## Unused Code Report

### Orphaned Files (not imported/referenced anywhere)
| File | Last Modified | Probable Reason |
|------|--------------|-----------------|
| util/DateUtil.java | 2024-01-15 | Legacy — replaced by java.time usage |
| model/dto/LegacyOrderResponse.java | 2023-11-20 | Old API version, not removed |

### Unused Functions (defined but never called)
| File | Line | Function | Visibility | Notes |
|------|------|----------|-----------|-------|
| service/UserService.java | 120 | `deactivateUser(Long id)` | public | No controller calls this — possible dead feature |
| util/JwtUtil.java | 85 | `refreshToken(String old)` | public | Refresh logic moved to AuthService |
| repository/UserRepository.java | 25 | `findByPhoneNumber(String phone)` | public | Phone login feature was abandoned |

### Unused Dependencies (in build file but not imported in code)
| Dependency | Build File | Line | Notes |
|-----------|-----------|------|-------|
| com.google.guava:guava:32.0 | pom.xml | 85 | No Guava imports found in source |
| io.micrometer:micrometer-registry-datadog | pom.xml | 102 | Only Prometheus registry is configured |

### Unused Configuration Properties
| Property | File | Line | Referenced By |
|----------|------|------|--------------|
| app.feature.legacy-export | application.yml | 45 | Nothing — no @Value or @ConfigurationProperties reads this |
```

### 3.3 API Endpoint Summary Table

```
## API Endpoint Summary

| # | Method | Path | Auth | Roles | Controller | Service | Tables | External Calls | Status |
|---|--------|------|------|-------|-----------|---------|--------|---------------|--------|
| 1 | POST | /api/v1/auth/login | No | — | AuthController:25 | AuthService:40 | users(R) | — | Active |
| 2 | POST | /api/v1/auth/register | No | — | AuthController:50 | AuthService:70 | users(W) | notification-svc | Active |
| 3 | GET | /api/v1/users/{id} | Yes | USER,ADMIN | UserController:30 | UserService:45 | users(R) | — | Active |
| 4 | PUT | /api/v1/users/{id} | Yes | ADMIN | UserController:60 | UserService:80 | users(RW) | — | Active |
| 5 | POST | /api/v1/orders | Yes | USER | OrderController:25 | OrderService:50 | orders(W), users(R) | payment-svc | Active |
| 6 | GET | /api/v1/orders/{id} | Yes | USER,ADMIN | OrderController:45 | OrderService:70 | orders(R) | — | Active |
```

### 3.4 Database Schema & Access Map

```
## Database Access Map

### Tables and Their Accessors
| Table | Entity | Repository | Read By (APIs) | Written By (APIs) | Indexes |
|-------|--------|-----------|---------------|-------------------|---------|
| users | User.java | UserRepository.java | GET /users/{id}, GET /users | POST /auth/register, PUT /users/{id} | PK(id), UNIQUE(email) |
| orders | Order.java | OrderRepository.java | GET /orders/{id}, GET /orders | POST /orders | PK(id), FK(user_id), IDX(status) |

### Migration History
| Version | File | Description | Tables Affected |
|---------|------|-------------|----------------|
| V1 | V1__create_users.sql | Initial users table | users |
| V2 | V2__create_orders.sql | Orders table with FK to users | orders |
```

### 3.5 Configuration & Environment Surface

```
## Configuration Surface

### All Environment Variables / Config Properties
| Property | Source | Type | Default | Used By | Sensitive |
|----------|--------|------|---------|---------|-----------|
| server.port | application.yml | int | 8080 | Embedded Tomcat | no |
| spring.datasource.url | env: DB_URL | string | — | DataSource | yes |
| security.jwt.secret | Vault | string | — | JwtUtil | yes |
| app.rate-limit.rpm | application.yml | int | 100 | RateLimitFilter | no |

### Feature Flags
| Flag | Type | Default | Controlled By | Affects APIs |
|------|------|---------|--------------|-------------|
| app.feature.new-checkout | boolean | false | env var | POST /orders |
```

### 3.6 Error Handling Architecture

```
## Error Handling Architecture

### Global Exception Handler
- **File:** exception/GlobalExceptionHandler.java
- **Pattern:** @ControllerAdvice with @ExceptionHandler methods

### Exception Hierarchy
| Exception | Extends | HTTP Status | Thrown By |
|-----------|---------|-------------|----------|
| ResourceNotFoundException | RuntimeException | 404 | All services |
| DuplicateResourceException | RuntimeException | 409 | UserService, OrderService |
| UnauthorizedException | RuntimeException | 401 | JwtAuthFilter |
| ValidationException | RuntimeException | 400 | Custom validators |
| (unhandled) | Exception | 500 | GlobalExceptionHandler fallback |

### Error Response Format
```json
{
  "timestamp": "ISO-8601",
  "status": 404,
  "error": "Not Found",
  "message": "User with id 42 not found",
  "path": "/api/v1/users/42",
  "traceId": "abc-123-def"
}
```
```

### 3.7 Security Architecture

```
## Security Architecture

### Authentication Flow
1. Client sends POST /api/v1/auth/login with credentials
2. AuthService validates credentials against DB (BCrypt compare)
3. On success: returns JWT access token (15 min) + refresh token (7 days)
4. Client sends Bearer token in Authorization header on subsequent requests
5. JwtAuthFilter extracts, validates, sets SecurityContext
6. Spring Security checks role-based access rules per endpoint

### Filter Chain Order
| Priority | Filter | Purpose |
|----------|--------|---------|
| 1 | RequestLoggingFilter | Log all requests |
| 2 | RateLimitFilter | Throttle abusive clients |
| 3 | JwtAuthFilter | Token validation |
| 4 | SecurityFilterChain | Authorization rules |

### Endpoint Security Matrix
| Endpoint | Public | Authenticated | ADMIN | USER | Notes |
|----------|--------|--------------|-------|------|-------|
| POST /auth/login | ✓ | — | — | — | Rate limited: 5/min |
| POST /auth/register | ✓ | — | — | — | Rate limited: 3/min |
| GET /users/{id} | — | ✓ | ✓ | ✓ (own only) | @PreAuthorize check |
| PUT /users/{id} | — | ✓ | ✓ | — | Admin only |
| POST /orders | — | ✓ | ✓ | ✓ | — |
```

---

## Phase 4: Architecture Quality Assessment

### 4.1 Complexity Hotspots

```
## Complexity Hotspots

### Files by Responsibility Count
| File | Lines | Functions | Injected Dependencies | Responsibility Count | Verdict |
|------|-------|-----------|----------------------|---------------------|---------|
| service/OrderService.java | 450 | 22 | 8 | 5 (CRUD + payment + notification + audit + validation) | **Split recommended** |
| controller/UserController.java | 120 | 6 | 2 | 1 (HTTP handling) | OK |

### Longest Call Chains
| API | Chain Depth | Deepest Path |
|-----|-------------|-------------|
| POST /orders | 9 | Controller → OrderService → PaymentClient → (HTTP) → PaymentService → StripeClient |
| POST /auth/register | 6 | Controller → AuthService → UserRepository → (DB) → NotificationClient → (HTTP) |
```

### 4.2 Duplicated Logic

```
## Duplicated Logic

| Pattern | Occurrences | Files | Suggested Extraction |
|---------|------------|-------|---------------------|
| Null-check + throw NotFoundException | 12 | UserService:45, UserService:60, OrderService:30, ... | Extract to `EntityLookup.findOrThrow(repo, id)` |
| JWT token extraction from header | 3 | JwtAuthFilter:35, WebSocketAuthHandler:20, TestHelper:15 | Already in JwtUtil — two callers don't use it |
```

### 4.3 Test Coverage Gaps

```
## Test Coverage Gaps

### APIs Without Corresponding Tests
| API | Controller | Test File Exists | Integration Test | Unit Test |
|-----|-----------|-----------------|-----------------|-----------|
| POST /orders | OrderController.java | ✗ | ✗ | ✗ |
| PUT /users/{id} | UserController.java | ✓ | ✗ | ✓ |

### Services Without Tests
| Service | Functions | Tested Functions | Gap |
|---------|-----------|-----------------|-----|
| OrderService | 22 | 8 | 14 functions untested |
```

---

## Output Requirements

1. **Every file path must be relative to the project root.**
2. **Every function reference must include file:line.**
3. **Every table in this document must be populated — no placeholder "N/A" rows unless genuinely not applicable.**
4. **The output must be written to a single markdown file** (e.g., `docs/service-map.md` or as specified by the user).
5. **Sections must appear in the exact order defined above** — Phase 1, Phase 2 (per API), Phase 3, Phase 4.
6. **If the service has custom in-repo libraries**, Phase 1.2 must include a separate file tree for each library, and Phase 2 call chains must trace into them.
7. **If an API endpoint has no middleware**, the middleware table should state "None — no filters configured for this path."
8. **If there are no unused functions**, the unused code report should explicitly state "No unused code detected" rather than omitting the section.

---

## How to Execute This Skill

### Step-by-step execution protocol:

1. **Discover entrypoints** — find the main application class/function and all route/controller registrations.
2. **List all API endpoints** — scan controllers, route files, or framework-specific decorators to build the complete endpoint list.
3. **For each endpoint, trace the full call chain** — follow every function call from the handler to the deepest layer. Read every file. Do not assume.
4. **Identify custom libraries** — check if any imports resolve to other modules/packages within the same repository (monorepo sibling, `libs/` directory, workspace package).
5. **Trace into custom libraries** — for every call that crosses into a custom library, read the target file and document the function signature, purpose, and any further calls it makes.
6. **Map database operations** — for every repository/data access call, identify the table, operation type (R/W), and the query or method.
7. **Map external calls** — for every HTTP client, gRPC stub, message producer, or event publisher, document the target service, protocol, timeout, and retry config.
8. **Scan for unused code** — after all call chains are mapped, identify files not referenced in any chain, functions not called by any other function, and dependencies not imported.
9. **Build the cross-cutting sections** — dependency graph, security architecture, error handling, configuration surface.
10. **Assess quality** — identify complexity hotspots, duplicated logic, and test coverage gaps.

### What NOT to do:
- Do not guess function signatures — read the actual code.
- Do not skip "boring" endpoints like health checks — map everything.
- Do not stop at interface boundaries — follow to the concrete implementation.
- Do not trust existing documentation — verify against the code.
- Do not omit sections because the service is small — a small service still gets the full template.
