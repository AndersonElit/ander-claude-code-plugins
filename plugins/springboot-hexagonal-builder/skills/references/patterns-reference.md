# Patterns Reference

Detailed catalog of design patterns and their application in Java hexagonal architecture projects. This file is loaded on-demand — consult it when you need specifics about a pattern.

## Table of Contents

1. [GoF Creational Patterns](#gof-creational-patterns)
2. [GoF Structural Patterns](#gof-structural-patterns)
3. [GoF Behavioral Patterns](#gof-behavioral-patterns)
4. [Microservice Patterns](#microservice-patterns)
5. [Exception Handling Strategy](#exception-handling-strategy)
6. [Reactive Programming Guidelines](#reactive-programming-guidelines)

---

## GoF Creational Patterns

| Pattern | When to use | Example in hexagonal context |
|---------|-------------|------------------------------|
| **Builder** | Objects with many optional params, complex construction | Domain entities with many fields, complex DTOs. Use Lombok `@Builder`. |
| **Factory Method** | Object creation varies by input/config | Creating different adapter implementations based on config (e.g., `NotificationSenderFactory` returning email/SMS/push) |
| **Abstract Factory** | Families of related objects | Database adapter factories producing matching repository + entity + mapper sets |
| **Singleton** | Single shared instance (use sparingly) | Spring beans are singletons by default — rarely need explicit Singleton pattern |

### Builder Example
```java
// Domain entity — use Lombok @Builder for clean construction
@Data
@Builder
public class Order {
    private String id;
    private String customerId;
    private List<OrderItem> items;
    private OrderStatus status;
    private BigDecimal totalAmount;
}

// Usage
Order order = Order.builder()
    .customerId("cust-123")
    .items(items)
    .status(OrderStatus.PENDING)
    .totalAmount(calculateTotal(items))
    .build();
```

### Factory Example
```java
// Port in domain
public interface NotificationSender {
    Mono<Void> send(Notification notification);
}

// Factory in application layer (returns port interface, not implementation)
@Component
@RequiredArgsConstructor
public class NotificationSenderFactory {
    private final Map<String, NotificationSender> senders;

    public NotificationSender getSender(NotificationType type) {
        return senders.get(type.name().toLowerCase());
    }
}
```

---

## GoF Structural Patterns

| Pattern | When to use | Example in hexagonal context |
|---------|-------------|------------------------------|
| **Adapter** | Convert between domain and infrastructure interfaces | Repository adapters are literally this pattern — they adapt the domain port to a framework-specific implementation |
| **Decorator** | Add behavior without modifying the class | Logging/caching/metrics around use cases: `LoggingUserUseCase` wrapping `UserUseCaseImpl` |
| **Facade** | Simplify a complex subsystem | A use case that orchestrates multiple ports is already a facade |
| **Proxy** | Control access or add lazy behavior | Spring's `@Transactional` is a proxy. Resilience4j circuit breakers wrap adapters as proxies. |

### Decorator Example
```java
// Logging decorator for a use case
@Component
@Primary
@RequiredArgsConstructor
public class LoggingOrderUseCase implements OrderUseCase {
    private final OrderUseCaseImpl delegate;
    private static final Logger log = LoggerFactory.getLogger(LoggingOrderUseCase.class);

    @Override
    public Mono<Order> create(Order order) {
        return delegate.create(order)
            .doOnSubscribe(s -> log.info("Creating order for customer: {}", order.getCustomerId()))
            .doOnSuccess(o -> log.info("Order created: {}", o.getId()))
            .doOnError(e -> log.error("Order creation failed", e));
    }
}
```

---

## GoF Behavioral Patterns

| Pattern | When to use | Example in hexagonal context |
|---------|-------------|------------------------------|
| **Strategy** | Multiple interchangeable algorithms | Payment processing (`CreditCardStrategy`, `PayPalStrategy`), pricing rules, validation strategies |
| **Observer** | React to events without coupling | Domain events: `OrderCreatedEvent` notified to inventory, notifications, analytics |
| **Template Method** | Common algorithm skeleton with variable steps | Base use case with hooks: `validate() → execute() → notify()` |
| **Chain of Responsibility** | Processing pipeline where each step may handle or pass | Request validation chains, middleware-like processing |
| **Command** | Encapsulate operations as objects | CQRS commands: `CreateOrderCommand`, `UpdateStatusCommand` |

### Strategy Example
```java
// Strategy interface (port in domain)
public interface PricingStrategy {
    Mono<BigDecimal> calculatePrice(Order order);
}

// Implementations (driven-adapters or application layer)
@Component("standard")
public class StandardPricing implements PricingStrategy {
    public Mono<BigDecimal> calculatePrice(Order order) {
        return Mono.just(order.getBaseAmount());
    }
}

@Component("premium")
public class PremiumPricing implements PricingStrategy {
    public Mono<BigDecimal> calculatePrice(Order order) {
        return Mono.just(order.getBaseAmount().multiply(new BigDecimal("0.9")));
    }
}

// Use case selects strategy
@Service
@RequiredArgsConstructor
public class OrderUseCaseImpl implements OrderUseCase {
    private final Map<String, PricingStrategy> pricingStrategies;

    public Mono<Order> create(Order order) {
        PricingStrategy strategy = pricingStrategies.get(order.getCustomerTier());
        return strategy.calculatePrice(order)
            .map(price -> order.toBuilder().totalAmount(price).build());
    }
}
```

### Observer/Event Example
```java
// Domain event
public record OrderCreatedEvent(String orderId, String customerId, BigDecimal amount) {}

// Port (outbound — domain says "I need to publish events")
public interface DomainEventPublisher {
    Mono<Void> publish(Object event);
}

// Use case publishes events through the port
@Service
@RequiredArgsConstructor
public class OrderUseCaseImpl implements OrderUseCase {
    private final OrderRepository repository;
    private final DomainEventPublisher eventPublisher;

    public Mono<Order> create(Order order) {
        return repository.save(order)
            .flatMap(saved -> eventPublisher
                .publish(new OrderCreatedEvent(saved.getId(), saved.getCustomerId(), saved.getTotalAmount()))
                .thenReturn(saved));
    }
}
```

---

## Microservice Patterns

| Pattern | Problem it solves | Implementation |
|---------|-------------------|----------------|
| **Circuit Breaker** | Prevent cascading failures when external service is down | Resilience4j `@CircuitBreaker` on adapter methods that call external services |
| **Retry** | Transient failures in external calls | Resilience4j `@Retry` or Reactor's `.retryWhen()` |
| **CQRS** | Read and write models have different optimization needs | Separate query and command use cases, potentially different data stores |
| **Saga** | Distributed transactions across services | Orchestration (central coordinator) or choreography (event-driven) |
| **Outbox** | Ensure event publishing is consistent with DB writes | Write event to outbox table in same DB transaction, poll/CDC to publish |
| **API Gateway** | Single entry point, cross-cutting concerns | Spring Cloud Gateway for routing, rate limiting, auth |
| **Event Sourcing** | Need complete audit trail or temporal queries | Store events instead of current state, rebuild state by replaying |

### Circuit Breaker Example
```java
// In a driven-adapter that calls an external service
@Component
@RequiredArgsConstructor
public class PaymentGatewayAdapter implements PaymentGateway {
    private final WebClient webClient;

    @CircuitBreaker(name = "payment", fallbackMethod = "paymentFallback")
    public Mono<PaymentResult> process(Payment payment) {
        return webClient.post()
            .uri("/payments")
            .bodyValue(payment)
            .retrieve()
            .bodyToMono(PaymentResult.class);
    }

    private Mono<PaymentResult> paymentFallback(Payment payment, Throwable t) {
        return Mono.just(PaymentResult.deferred(payment.getId(), "Service temporarily unavailable"));
    }
}
```

---

## Exception Handling Strategy

### Layer-specific exceptions

```java
// Domain layer — business rule violations
public class DomainException extends RuntimeException {
    public DomainException(String message) { super(message); }
}

public class EntityNotFoundException extends DomainException {
    public EntityNotFoundException(String entity, String id) {
        super(String.format("%s with id '%s' not found", entity, id));
    }
}

public class BusinessRuleException extends DomainException {
    public BusinessRuleException(String rule) {
        super(String.format("Business rule violated: %s", rule));
    }
}
```

### Global exception handler (entry-points/app)

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    public Mono<ResponseEntity<ErrorResponse>> handleNotFound(EntityNotFoundException ex) {
        return Mono.just(ResponseEntity.status(404)
            .body(new ErrorResponse("NOT_FOUND", ex.getMessage())));
    }

    @ExceptionHandler(BusinessRuleException.class)
    public Mono<ResponseEntity<ErrorResponse>> handleBusinessRule(BusinessRuleException ex) {
        return Mono.just(ResponseEntity.status(422)
            .body(new ErrorResponse("BUSINESS_RULE_VIOLATION", ex.getMessage())));
    }

    @ExceptionHandler(Exception.class)
    public Mono<ResponseEntity<ErrorResponse>> handleGeneral(Exception ex) {
        log.error("Unexpected error", ex);
        return Mono.just(ResponseEntity.status(500)
            .body(new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred")));
    }
}

public record ErrorResponse(String code, String message) {}
```

---

## Reactive Programming Guidelines

### Operator selection

| Need | Operator | Why |
|------|----------|-----|
| Transform one value | `.map()` | Synchronous transformation |
| Transform to another Mono/Flux | `.flatMap()` | Asynchronous chaining |
| Do something on the side | `.doOnNext()`, `.doOnSuccess()` | Logging, metrics (no side effects on the stream) |
| Handle errors | `.onErrorResume()`, `.onErrorMap()` | Convert or recover from errors |
| Provide default | `.switchIfEmpty()`, `.defaultIfEmpty()` | When Mono is empty |
| Combine results | `Mono.zip()`, `Flux.merge()` | Parallel or concurrent operations |
| Sequential dependent calls | `.flatMap()` chain | When B depends on A's result |

### Common mistakes to avoid

```java
// BAD: blocking in reactive chain
public Mono<User> getUser(String id) {
    User user = repository.findById(id).block(); // breaks non-blocking
    return Mono.just(user);
}

// GOOD: stay reactive
public Mono<User> getUser(String id) {
    return repository.findById(id);
}

// BAD: ignoring empty Mono
public Mono<User> getUser(String id) {
    return repository.findById(id); // returns empty Mono if not found — caller gets 200 with empty body
}

// GOOD: handle empty explicitly
public Mono<User> getUser(String id) {
    return repository.findById(id)
        .switchIfEmpty(Mono.error(new EntityNotFoundException("User", id)));
}

// BAD: nested subscribes
public void process(Order order) {
    repository.save(order).subscribe(saved -> {
        eventPublisher.publish(new OrderCreated(saved)).subscribe(); // nested subscribe = lost backpressure
    });
}

// GOOD: chain with flatMap
public Mono<Order> process(Order order) {
    return repository.save(order)
        .flatMap(saved -> eventPublisher.publish(new OrderCreated(saved))
            .thenReturn(saved));
}
```

### Testing reactive code

```java
@Test
void shouldReturnUserWhenExists() {
    // Arrange
    when(repository.findById("123")).thenReturn(Mono.just(testUser));

    // Act & Assert
    StepVerifier.create(useCase.getById("123"))
        .expectNext(testUser)
        .verifyComplete();
}

@Test
void shouldErrorWhenUserNotFound() {
    when(repository.findById("999")).thenReturn(Mono.empty());

    StepVerifier.create(useCase.getById("999"))
        .expectError(EntityNotFoundException.class)
        .verify();
}
```
