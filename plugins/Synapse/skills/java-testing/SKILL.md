---
name: java-testing
description: Enforces rigorous Java testing standards using JUnit 5, Mockito, and strict null-safety assertions.
---

# Java Testing Standards

All Java code MUST adhere to these testing mandates before it can be considered complete or merge-ready.

## Mandate 1: Use JUnit 5 for All Unit Tests

Every test class must use JUnit 5 (`org.junit.jupiter.api`). JUnit 4 annotations (`@org.junit.Test`, `@RunWith`) are not allowed. Use `@Nested` classes to group related test scenarios and `@DisplayName` for human-readable test descriptions.

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

@DisplayName("PaymentProcessor Tests")
class PaymentProcessorTest {

    @Nested
    @DisplayName("when processing valid payments")
    class ValidPayments {
        @Test
        @DisplayName("should debit the correct amount")
        void shouldDebitCorrectAmount() {
            // ...
        }
    }
}
```

## Mandate 2: Use Mockito for Mocking External Dependencies

All external dependencies (databases, HTTP clients, message queues, third-party services) MUST be mocked using Mockito. Never connect to real external systems in unit tests. Use `@Mock` and `@InjectMocks` annotations with `@ExtendWith(MockitoExtension.class)`.

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private PaymentGateway paymentGateway;

    @InjectMocks
    private OrderService orderService;

    @Test
    void shouldSaveOrderWhenPaymentSucceeds() {
        when(paymentGateway.charge(any())).thenReturn(PaymentResult.SUCCESS);
        orderService.placeOrder(new Order("item-1", 100));
        verify(orderRepository).save(any(Order.class));
    }
}
```

## Mandate 3: Strict Null-Safety Assertions for Edge Cases

Every test suite MUST include explicit edge case tests that guard against `NullPointerException`. Test with `null` inputs, empty collections, and missing optional fields. Use `assertThrows` for expected exceptions and `assertDoesNotThrow` for graceful handling.

```java
@Test
@DisplayName("should throw IllegalArgumentException for null input")
void shouldRejectNullInput() {
    assertThrows(IllegalArgumentException.class, () -> service.process(null));
}

@Test
@DisplayName("should return empty list for empty input, not null")
void shouldReturnEmptyListNotNull() {
    List<Result> results = service.process(Collections.emptyList());
    assertNotNull(results);
    assertTrue(results.isEmpty());
}
```
