# Framework-Specific Discovery Patterns

This reference helps the agent locate API endpoints, middleware, and configuration across different frameworks. When mapping a service, identify the framework first, then use the corresponding discovery pattern.

---

## Java / Spring Boot

### Finding Endpoints
```
Annotations to scan:
  @RestController, @Controller
  @GetMapping, @PostMapping, @PutMapping, @PatchMapping, @DeleteMapping
  @RequestMapping (class-level prefix + method-level path)

Route registration:
  RouterFunction<ServerResponse> beans (functional endpoints)
```

### Finding Middleware / Filters
```
Scan for:
  extends OncePerRequestFilter
  implements Filter (javax.servlet / jakarta.servlet)
  implements HandlerInterceptor
  @ControllerAdvice (exception handlers)
  SecurityFilterChain beans (Spring Security filter ordering)
  FilterRegistrationBean definitions

Ordering:
  @Order annotation or Ordered interface
  SecurityFilterChain.addFilterBefore / addFilterAfter
  FilterRegistrationBean.setOrder()
```

### Finding Services & Repositories
```
  @Service classes — business logic
  @Repository classes — data access
  @Component classes — general beans
  JpaRepository / CrudRepository / custom interfaces extending Spring Data
  @Query annotations for custom queries
  EntityManager direct usage for native queries
```

### Finding Configuration
```
  @Configuration classes
  @ConfigurationProperties records/classes
  @Value("${...}") injections
  application.yml / application.properties
  application-{profile}.yml for profile-specific config
  @Profile annotations for conditional beans
```

### Finding Custom Libraries
```
  Multi-module Maven: check parent pom.xml for <modules>
  Multi-module Gradle: check settings.gradle for include
  Local dependencies: check for project(":module-name") or <module> references
  Shared packages: imports from com.company.common.*, com.company.shared.*
```

---

## Python / FastAPI

### Finding Endpoints
```
Scan for:
  @app.get, @app.post, @app.put, @app.patch, @app.delete
  @router.get, @router.post, etc. (APIRouter instances)
  app.include_router() calls — connects routers to app

Route registration:
  APIRouter prefix parameter
  app.include_router(router, prefix="/api/v1")
```

### Finding Middleware
```
  app.add_middleware() calls
  @app.middleware("http") decorated functions
  Depends() — FastAPI dependency injection (acts as middleware)
  BackgroundTasks — async operations triggered post-response
  Starlette middleware classes (BaseHTTPMiddleware subclasses)
```

### Finding Services & Repositories
```
  Classes injected via Depends()
  Module-level service instances
  SQLAlchemy session factories and repository patterns
  Async database sessions (asyncpg, databases library)
```

### Finding Configuration
```
  pydantic BaseSettings subclasses
  .env files loaded by python-dotenv
  settings.py / config.py modules
  os.environ.get() calls
```

### Finding Custom Libraries
```
  pyproject.toml [tool.poetry.packages] or [tool.setuptools.packages]
  Monorepo: check for workspace definitions in pyproject.toml
  Local path dependencies: {path = "../common-lib"} in pyproject.toml
  Relative imports: from ...common import X
```

---

## Python / Django

### Finding Endpoints
```
  urlpatterns in urls.py files (path(), re_path())
  Include() for nested URL configs
  @api_view decorator (DRF)
  ViewSet and Router registration (DRF)
  ModelViewSet, GenericAPIView subclasses
```

### Finding Middleware
```
  MIDDLEWARE list in settings.py
  Custom middleware classes with __call__ or process_request/process_response
  DRF: permission_classes, authentication_classes, throttle_classes
  Decorators: @permission_required, @login_required
```

### Finding Services
```
  No standard location — check for services.py, domain.py, or use_cases.py in each app
  Manager classes on models (custom QuerySet methods)
  Signals (post_save, pre_delete, etc.)
  Celery tasks in tasks.py
```

### Finding Configuration
```
  settings.py / settings/*.py (split settings)
  django-environ or python-decouple for env vars
  DATABASES, CACHES, REST_FRAMEWORK dicts in settings
```

---

## TypeScript / Express

### Finding Endpoints
```
  app.get(), app.post(), app.put(), app.patch(), app.delete()
  router.get(), router.post(), etc.
  app.use("/prefix", router) — mounts routers
  Route files typically in routes/ or controllers/ directory
```

### Finding Middleware
```
  app.use() calls — global middleware
  router.use() — route-specific middleware
  Middleware functions: (req, res, next) => {}
  Error middleware: (err, req, res, next) => {}
  Common: cors(), helmet(), morgan(), express-rate-limit
```

### Finding Services & Repositories
```
  Classes or modules in services/ directory
  Database access patterns: Sequelize models, TypeORM repositories, Prisma client
  Dependency injection: if using tsyringe, inversify, or similar
```

### Finding Configuration
```
  .env files with dotenv
  config/ directory with environment-specific files
  process.env.VARIABLE_NAME usage
  convict, nconf, or custom config loaders
```

---

## TypeScript / NestJS

### Finding Endpoints
```
  @Controller() decorated classes
  @Get(), @Post(), @Put(), @Patch(), @Delete() method decorators
  @ApiTags() for Swagger grouping
  Module imports chain: AppModule → FeatureModule → Controller
```

### Finding Middleware / Guards / Interceptors / Pipes
```
  @UseGuards() — authentication and authorization
  @UseInterceptors() — logging, transformation, caching
  @UsePipes() — validation (ValidationPipe)
  implements NestMiddleware — HTTP middleware
  APP_GUARD, APP_INTERCEPTOR, APP_PIPE — global providers
  Execution order: Middleware → Guards → Interceptors (pre) → Pipes → Handler → Interceptors (post) → Exception Filters
```

### Finding Services & Repositories
```
  @Injectable() decorated classes
  @InjectRepository() for TypeORM
  PrismaService for Prisma
  Module providers array
```

---

## Go / Gin

### Finding Endpoints
```
  r.GET(), r.POST(), r.PUT(), r.PATCH(), r.DELETE()
  r.Group("/prefix") — route groups
  router.Handle() for custom methods
  Handler functions: func(c *gin.Context)
```

### Finding Middleware
```
  r.Use() — global middleware
  group.Use() — group-level middleware
  Middleware functions: func(c *gin.Context) with c.Next()
  Common: gin.Logger(), gin.Recovery(), cors.Default()
```

---

## Go / Standard Library (net/http)

### Finding Endpoints
```
  http.HandleFunc("/path", handler)
  mux.Handle(), mux.HandleFunc()
  gorilla/mux: r.HandleFunc("/path").Methods("GET")
  chi: r.Get("/path", handler), r.Route("/prefix", func(r chi.Router) {...})
```

### Finding Middleware
```
  Handler wrapping: middleware(nextHandler)
  chi: r.Use(middleware)
  Alice chain: alice.New(m1, m2, m3).Then(handler)
```

---

## Rust / Actix-web

### Finding Endpoints
```
  #[get("/path")], #[post("/path")], etc.
  web::resource("/path").route(web::get().to(handler))
  App::new().service() and .configure() calls
  scope("/prefix") for grouped routes
```

### Finding Middleware
```
  App::new().wrap() — global middleware
  web::scope().wrap() — scope-level
  Transform trait implementations
  Common: Logger, Cors, identity middleware
```

---

## General Discovery Strategies (Any Framework)

### Finding Custom In-Repo Libraries
1. **Check build manifests** for local/workspace dependencies:
   - Maven: `<module>` in parent pom
   - Gradle: `include` in settings.gradle, `project(":lib")` dependencies
   - npm/pnpm: `workspace:*` in package.json
   - Poetry: `{path = "../lib"}` in pyproject.toml
   - Go: `replace` directives in go.mod
   - Cargo: `path = "../lib"` in Cargo.toml

2. **Check import paths** that resolve to sibling directories rather than `node_modules`, `site-packages`, or Maven Central.

3. **Check for a `libs/`, `packages/`, `modules/`, or `shared/` directory** at the repo root.

### Finding Unused Code
1. **Unused files**: Build an import graph from all source files. Any file not imported (directly or transitively) from an entrypoint is orphaned.
2. **Unused functions**: For every public/exported function, grep for callers. If only defined and never called, it's unused.
3. **Unused dependencies**: For every dependency in the build manifest, check if any source file imports from it.
4. **Unused config**: For every config property, check if any source file reads it via environment variable access, config injection, or property binding.

### Finding Database Interactions
1. **ORM patterns**: Repository interfaces, model definitions, migration files
2. **Raw queries**: String literals containing SQL keywords (SELECT, INSERT, UPDATE, DELETE)
3. **Query builders**: Method chains like `.where()`, `.select()`, `.join()`
4. **Connection config**: Database URLs, connection pool settings

### Finding External Service Calls
1. **HTTP clients**: RestTemplate, WebClient, HttpClient, requests, axios, fetch, http.Client
2. **gRPC stubs**: Generated client classes, .proto files
3. **Message producers**: KafkaTemplate, channel.sendToQueue, boto3 SQS client
4. **SDK clients**: AWS SDK, GCP client libraries, Stripe, Twilio, etc.
