# System Design Patterns Reference

Concrete implementation patterns the service-surgeon agent applies when modifying codebases. Each pattern includes when to use it, when NOT to use it, and implementation across common frameworks.

---

## Construction Patterns

### Factory Pattern (Mandatory)

All domain object construction MUST go through factories. No exceptions.

#### Simple Factory
```java
@Component
public class OrderFactory {
    public Order create(OrderRequest req, User user) {
        return Order.builder()
            .userId(user.getId())
            .items(req.getItems().stream().map(this::toOrderItem).toList())
            .status(OrderStatus.PENDING)
            .totalAmount(calculateTotal(req.getItems()))
            .createdAt(Instant.now())
            .build();
    }

    private OrderItem toOrderItem(OrderItemRequest req) {
        return OrderItem.builder()
            .productId(req.getProductId())
            .quantity(req.getQuantity())
            .unitPrice(req.getUnitPrice())
            .build();
    }

    private BigDecimal calculateTotal(List<OrderItemRequest> items) {
        return items.stream()
            .map(i -> i.getUnitPrice().multiply(BigDecimal.valueOf(i.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

#### Abstract Factory — When You Need Families of Related Objects
```java
public interface NotificationFactory {
    NotificationSender createSender();
    NotificationTemplate createTemplate(NotificationType type);
    NotificationLogger createLogger();
}

@Component
@Profile("production")
public class ProductionNotificationFactory implements NotificationFactory {
    @Override public NotificationSender createSender() { return new SesEmailSender(sesClient); }
    @Override public NotificationTemplate createTemplate(NotificationType type) { return new S3Template(s3Client, type); }
    @Override public NotificationLogger createLogger() { return new CloudWatchLogger(cwClient); }
}

@Component
@Profile("test")
public class TestNotificationFactory implements NotificationFactory {
    @Override public NotificationSender createSender() { return new InMemorySender(); }
    @Override public NotificationTemplate createTemplate(NotificationType type) { return new StaticTemplate(type); }
    @Override public NotificationLogger createLogger() { return new ConsoleLogger(); }
}
```

#### Factory Method — When Subclasses Decide Construction
```java
public abstract class PaymentProcessorFactory {
    public abstract PaymentProcessor create(PaymentMethod method);

    // Template method — common validation + factory-specific creation
    public final PaymentProcessor createAndValidate(PaymentMethod method) {
        PaymentProcessor processor = create(method);
        processor.validateConfiguration();
        return processor;
    }
}

@Component
public class DefaultPaymentProcessorFactory extends PaymentProcessorFactory {
    private final Map<PaymentMethod, PaymentProcessor> processors;

    @Override
    public PaymentProcessor create(PaymentMethod method) {
        return Optional.ofNullable(processors.get(method))
            .orElseThrow(() -> new UnsupportedPaymentMethodException(method));
    }
}
```

---

## Migration Patterns

### Strangler Fig

Incrementally replace legacy code by routing new requests through new code while old code handles existing requests.

```
Phase 1: Deploy new service alongside old
  [Client] → [API Gateway] → [Legacy Service]

Phase 2: Route new endpoints to new service
  [Client] → [API Gateway] → [New Service]     ← new endpoints
                            → [Legacy Service]   ← existing endpoints

Phase 3: Migrate existing endpoints one by one
  [Client] → [API Gateway] → [New Service]     ← migrated + new
                            → [Legacy Service]   ← remaining only

Phase 4: Decommission legacy
  [Client] → [API Gateway] → [New Service]     ← everything
```

**Implementation via API Gateway routing:**
```yaml
# Kong / AWS API Gateway / Nginx config pattern
routes:
  - path: /api/v2/orders/*       # New version → new service
    service: order-service-v2
  - path: /api/v1/orders/*       # Old version → legacy (temporary)
    service: order-service-legacy
```

### Branch by Abstraction

Swap implementations behind an interface without breaking callers.

```java
// Step 1: Extract interface from existing code
public interface PaymentGateway {
    PaymentResult charge(PaymentRequest req);
    PaymentResult refund(String transactionId, BigDecimal amount);
}

// Step 2: Make existing code implement it
@Component("legacyPaymentGateway")
public class LegacyPaymentGateway implements PaymentGateway {
    // existing implementation unchanged
}

// Step 3: Build new implementation
@Component("stripePaymentGateway")
public class StripePaymentGateway implements PaymentGateway {
    // new implementation
}

// Step 4: Feature flag to switch
@Configuration
public class PaymentConfig {
    @Bean
    public PaymentGateway paymentGateway(
            @Value("${feature.new-payment-gateway:false}") boolean useNew,
            LegacyPaymentGateway legacy,
            StripePaymentGateway stripe) {
        return useNew ? stripe : legacy;
    }
}

// Step 5: After validation, remove legacy and feature flag
```

---

## Resilience Patterns

### Circuit Breaker

```java
@Service
public class PaymentClient {
    private final CircuitBreaker circuitBreaker;
    private final RestClient restClient;

    public PaymentResult charge(PaymentRequest req) {
        return circuitBreaker.executeSupplier(() ->
            restClient.post()
                .uri("/api/v1/payments/charge")
                .body(req)
                .retrieve()
                .body(PaymentResult.class)
        );
    }
}

// Configuration
@Bean
public CircuitBreaker paymentCircuitBreaker(CircuitBreakerRegistry registry) {
    return registry.circuitBreaker("payment", CircuitBreakerConfig.custom()
        .failureRateThreshold(50)                    // Open after 50% failure rate
        .waitDurationInOpenState(Duration.ofSeconds(30)) // Wait 30s before half-open
        .slidingWindowSize(10)                       // Evaluate last 10 calls
        .minimumNumberOfCalls(5)                     // Need at least 5 calls to evaluate
        .build());
}
```

### Retry with Exponential Backoff
```java
@Bean
public Retry paymentRetry(RetryRegistry registry) {
    return registry.retry("payment", RetryConfig.custom()
        .maxAttempts(3)
        .waitDuration(Duration.ofMillis(500))
        .intervalFunction(IntervalFunction.ofExponentialBackoff(500, 2.0))
        .retryOnException(e -> e instanceof ConnectException || e instanceof TimeoutException)
        .ignoreExceptions(BusinessException.class) // Don't retry business errors
        .build());
}
```

### Bulkhead — Isolate Failure Domains
```java
// Separate thread pools for separate dependencies
@Bean
public Bulkhead paymentBulkhead(BulkheadRegistry registry) {
    return registry.bulkhead("payment", BulkheadConfig.custom()
        .maxConcurrentCalls(10)        // Max 10 concurrent calls to payment service
        .maxWaitDuration(Duration.ofMillis(500)) // Queue for 500ms then reject
        .build());
}
```

---

## Data Patterns

### Outbox Pattern — Atomic DB Write + Event Publish

```java
// Problem: Need to save order AND publish event atomically
// Solution: Write event to outbox table in same transaction, publish asynchronously

@Transactional
public Order createOrder(OrderRequest req) {
    Order order = orderFactory.create(req);
    orderRepository.save(order);

    // Write to outbox in same transaction
    outboxRepository.save(OutboxEvent.builder()
        .aggregateType("Order")
        .aggregateId(order.getId().toString())
        .eventType("OrderCreated")
        .payload(objectMapper.writeValueAsString(order))
        .createdAt(Instant.now())
        .published(false)
        .build());

    return order;
}

// Separate publisher polls outbox and publishes (Debezium or scheduled task)
@Scheduled(fixedDelay = 1000)
@Transactional
public void publishOutboxEvents() {
    List<OutboxEvent> pending = outboxRepository.findByPublishedFalse();
    for (OutboxEvent event : pending) {
        kafkaTemplate.send(event.getEventType(), event.getPayload());
        event.setPublished(true);
    }
}
```

### CQRS — Separate Read and Write Models

```java
// Write side — domain model with business rules
@Service
public class OrderCommandService {
    public void createOrder(CreateOrderCommand cmd) {
        Order order = orderFactory.create(cmd);
        orderRepository.save(order);
        eventPublisher.publish(new OrderCreatedEvent(order));
    }
}

// Read side — optimized query model
@Service
public class OrderQueryService {
    private final OrderReadRepository readRepo; // Denormalized view

    public OrderSummaryDto getOrderSummary(Long orderId) {
        return readRepo.findSummaryById(orderId); // Pre-joined, no N+1
    }

    public Page<OrderListDto> listOrders(OrderFilter filter, Pageable pageable) {
        return readRepo.findAll(filter.toSpec(), pageable); // Optimized for listing
    }
}

// Event handler keeps read model in sync
@EventListener
public void onOrderCreated(OrderCreatedEvent event) {
    orderReadRepository.save(OrderReadModel.from(event.getOrder()));
}
```

### Saga — Distributed Transaction Coordination

```java
// Orchestration-based saga for multi-service transactions
@Service
public class OrderSaga {
    public void execute(CreateOrderCommand cmd) {
        // Step 1: Reserve inventory
        InventoryReservation reservation = inventoryClient.reserve(cmd.getItems());
        try {
            // Step 2: Charge payment
            PaymentResult payment = paymentClient.charge(cmd.getPaymentDetails());
            try {
                // Step 3: Create order
                orderService.createOrder(cmd, reservation, payment);
            } catch (Exception e) {
                // Compensate: refund payment
                paymentClient.refund(payment.getTransactionId());
                throw e;
            }
        } catch (Exception e) {
            // Compensate: release inventory
            inventoryClient.release(reservation.getId());
            throw e;
        }
    }
}
```

---

## Zero-Downtime Database Migration Patterns

### Add Column (Safe)
```sql
-- Step 1: Add nullable column
ALTER TABLE orders ADD COLUMN shipping_address_id BIGINT;

-- Step 2: Deploy code that writes to new column (but still reads old)

-- Step 3: Backfill existing rows
UPDATE orders SET shipping_address_id = (
    SELECT id FROM addresses WHERE addresses.order_id = orders.id
);

-- Step 4: Deploy code that reads from new column

-- Step 5: Add NOT NULL constraint (if needed)
ALTER TABLE orders ALTER COLUMN shipping_address_id SET NOT NULL;

-- Step 6: Drop old column (if replaced)
ALTER TABLE orders DROP COLUMN shipping_address;
```

### Rename Column (Safe)
```sql
-- NEVER: ALTER TABLE orders RENAME COLUMN old_name TO new_name;
-- This breaks running code instantly.

-- Instead:
-- Step 1: Add new column
ALTER TABLE orders ADD COLUMN new_name VARCHAR(255);

-- Step 2: Backfill
UPDATE orders SET new_name = old_name;

-- Step 3: Deploy code that writes to BOTH columns, reads from new

-- Step 4: Drop old column
ALTER TABLE orders DROP COLUMN old_name;
```

### Add Index (Safe)
```sql
-- ALWAYS use CONCURRENTLY to avoid table locks
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders (user_id);
```

---

## Anti-Patterns to Reject

| Anti-Pattern | Why | Fix |
|-------------|-----|-----|
| `new Service()` in controller | Bypasses DI, untestable | Constructor injection |
| God class (500+ lines) | Violates SRP, untestable | Extract into focused services |
| Business logic in controller | Tight coupling | Move to service layer |
| `@Autowired` on field | Hidden dependency | Constructor injection |
| Catching `Exception` globally | Swallows important errors | Catch specific exceptions |
| Returning `null` from service | Forces null checks everywhere | Return `Optional` or throw |
| `SELECT *` | Fetches unnecessary data, breaks on schema change | Select explicit columns |
| N+1 queries | Performance killer | Use JOIN FETCH or batch loading |
| Hardcoded config values | Unreproducible environments | Externalize to config/env |
| Mutable shared state in beans | Thread safety violations | Stateless design or synchronization |
| String concatenation in SQL | SQL injection vector | Parameterized queries |
| `Thread.sleep()` in production code | Blocks threads, unpredictable | Use async/scheduled mechanisms |
| Synchronous external calls in request path | Blocks thread pool under load | Async with circuit breaker |
