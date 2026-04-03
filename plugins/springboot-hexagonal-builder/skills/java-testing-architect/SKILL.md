---
name: java-testing-architect
description: "Generate, review, and refactor tests for Java microservices following hexagonal architecture testing best practices. Use this skill whenever the user asks to write unit tests, integration tests, create test classes, review test quality, add test coverage, write tests for a use case or adapter, set up Testcontainers, configure WireMock stubs, improve test assertions, apply AAA pattern, check test isolation, refactor existing tests, generate tests for Spring Boot / WebFlux / reactive services, or asks about Java testing strategy, test architecture, or test pyramid. Also trigger when the user says: escribir tests, crear pruebas unitarias, pruebas de integracion, generar tests, mejorar cobertura, revisar calidad de tests, configurar testcontainers, tests para microservicios."
---

# Java Backend Testing Architect

Generate and maintain high-quality test suites for Java microservices, ensuring every test is efficient, maintainable, and aligned with hexagonal architecture boundaries.

## How to Use This Skill

This skill supports four actions. Determine which one the user needs:

| Action | Trigger phrases | What to do |
|--------|----------------|------------|
| **generate** | "write tests", "create tests", "add tests for", "test this class" | Generate test classes following all rules below |
| **review** | "review tests", "check test quality", "are these tests good" | Audit existing tests against the standards below |
| **refactor** | "refactor tests", "improve tests", "clean up tests" | Fix violations and modernize test code |
| **strategy** | "testing strategy", "how should I test", "test architecture" | Recommend which test types to write and where |

If unclear, default to **generate** when the user points at production code, or **review** when they point at test code.

---

## Core Principle: Test Boundaries Follow Architecture Boundaries

In hexagonal architecture, each layer has a distinct testing strategy. The test type is determined by what you're testing, not by preference:

```
domain/model          --> Unit Tests (pure logic, zero dependencies)
application/use-cases --> Unit Tests (mock ports only)
driven-adapters/*     --> Integration Tests (real infra via Testcontainers/WireMock)
entry-points/rest-api --> Integration Tests (WebTestClient + mocked use cases)
entry-points/rabbit   --> Integration Tests (Testcontainers RabbitMQ)
```

This mapping exists because the whole point of hexagonal architecture is that domain and application logic are infrastructure-free. If you need Spring context to test a use case, something is wrong with the production code — the test is exposing a design flaw, not a test tooling problem.

---

## Unit Tests: Strict Isolation Policy

Unit tests validate domain logic and use case orchestration. They must be fast, deterministic, and free of framework context.

### What NOT to use in unit tests

Never use `@SpringBootTest`, `@WebFluxTest`, `@DataR2dbcTest`, or any annotation that boots a Spring context. These annotations start the application context, which makes the test slow and tests framework wiring rather than business logic. If a unit test needs Spring to run, the production code has a dependency problem that should be fixed first.

### Setup pattern

Use `@ExtendWith(MockitoExtension.class)` with `@Mock` for direct dependencies and `@InjectMocks` for the subject under test (SUT):

```java
@ExtendWith(MockitoExtension.class)
class CreateOrderUseCaseImplTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private EventPublisher eventPublisher;

    @InjectMocks
    private CreateOrderUseCaseImpl useCase;
}
```

Only mock **direct dependencies** of the SUT — these are the interfaces (ports) it receives via constructor injection. Never mock internal classes, static methods, or transitive dependencies. If the SUT has too many dependencies to mock comfortably (more than 4-5), that's an SRP signal — suggest splitting the class.

### Domain object preparation

Build domain objects using constructors or static factory methods — not reflection, not Spring beans. For complex objects with many fields, create a **Test Data Builder** to improve readability and avoid brittle constructors:

```java
class OrderTestBuilder {
    private String id = "order-001";
    private String customerId = "customer-001";
    private OrderStatus status = OrderStatus.PENDING;
    private List<OrderItem> items = List.of(defaultItem());

    static OrderTestBuilder anOrder() {
        return new OrderTestBuilder();
    }

    OrderTestBuilder withStatus(OrderStatus status) {
        this.status = status;
        return this;
    }

    OrderTestBuilder withItems(List<OrderItem> items) {
        this.items = items;
        return this;
    }

    Order build() {
        return new Order(id, customerId, status, items);
    }
}
```

Use builders when a domain entity has 4+ fields or when tests need many variations of the same object. For simple entities (2-3 fields), direct constructor calls are fine — don't over-engineer.

---

## Integration Tests: Infrastructure Fidelity

Integration tests verify that adapters correctly talk to real infrastructure. The key insight: an adapter test that uses H2 instead of PostgreSQL, or Fongo instead of MongoDB, is testing a fiction. Schema differences, query dialect, and driver behavior all differ between the fake and the real thing.

### Database adapters: Testcontainers

Use Testcontainers to spin up the real database engine. This catches schema issues, query compatibility problems, and driver-specific behavior that in-memory substitutes hide.

**PostgreSQL (R2DBC) adapter example:**

```java
@DataR2dbcTest
@Testcontainers
class OrderRepositoryAdapterIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.r2dbc.url", () ->
                String.format("r2dbc:postgresql://%s:%d/%s",
                        postgres.getHost(), postgres.getFirstMappedPort(), "testdb"));
        registry.add("spring.r2dbc.username", postgres::getUsername);
        registry.add("spring.r2dbc.password", postgres::getPassword);
    }

    @Autowired
    private OrderRepositoryAdapter adapter;

    @BeforeEach
    void cleanUp() {
        // Clean state before each test to guarantee idempotency
        adapter.deleteAll().block();
    }
}
```

**MongoDB adapter example:**

```java
@DataMongoTest
@Testcontainers
class ProductRepositoryAdapterIntegrationTest {

    @Container
    static MongoDBContainer mongo = new MongoDBContainer("mongo:7.0");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.mongodb.uri", mongo::getReplicaSetUrl);
    }

    @Autowired
    private ProductRepositoryAdapter adapter;

    @BeforeEach
    void cleanUp() {
        adapter.deleteAll().block();
    }
}
```

### External API clients: WireMock

Use WireMock to simulate third-party APIs. The critical rule: **always verify stubs were actually called** using `WireMock.verify()`. Without verification, a refactoring that accidentally removes the HTTP call will pass silently — the stub just never gets hit and the test still passes.

```java
@WireMockTest(httpPort = 8089)
class PaymentGatewayAdapterIntegrationTest {

    private PaymentGatewayAdapter adapter;

    @BeforeEach
    void setUp() {
        WebClient webClient = WebClient.builder()
                .baseUrl("http://localhost:8089")
                .build();
        adapter = new PaymentGatewayAdapter(webClient);
    }

    @Test
    void should_process_payment_successfully() {
        // Arrange
        stubFor(post(urlEqualTo("/api/payments"))
                .willReturn(aResponse()
                        .withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"transactionId": "txn-001", "status": "APPROVED"}
                                """)));

        PaymentRequest request = new PaymentRequest("100.00", "USD");

        // Act
        PaymentResponse response = adapter.processPayment(request).block();

        // Assert
        assertThat(response.transactionId()).isEqualTo("txn-001");
        assertThat(response.status()).isEqualTo("APPROVED");

        // Verify the external call was actually made
        verify(postRequestedFor(urlEqualTo("/api/payments"))
                .withHeader("Content-Type", equalTo("application/json")));
    }
}
```

### REST API controllers: WebTestClient

Test controllers with `@WebFluxTest` and mock the use case layer. This validates request/response mapping, status codes, and error handling without touching infrastructure:

```java
@WebFluxTest(OrderController.class)
class OrderControllerIntegrationTest {

    @Autowired
    private WebTestClient webTestClient;

    @MockBean
    private CreateOrderUseCase createOrderUseCase;

    @Test
    void should_return_201_when_order_created() {
        // Arrange
        Order order = anOrder().build();
        when(createOrderUseCase.execute(any())).thenReturn(Mono.just(order));

        // Act & Assert
        webTestClient.post().uri("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .bodyValue("""
                        {"customerId": "customer-001", "items": []}
                        """)
                .exchange()
                .expectStatus().isCreated()
                .expectBody()
                .jsonPath("$.id").isEqualTo("order-001");
    }
}
```

### State management

Every integration test must be idempotent — running it N times produces the same result. Two approaches:

1. **`@BeforeEach` cleanup** (preferred for Testcontainers): explicitly delete or reset state before each test. This is visible and predictable.
2. **`@Transactional`** (for JPA/JDBC): Spring rolls back after each test. Be careful with R2DBC — `@Transactional` behavior is less predictable with reactive repositories, so prefer explicit cleanup.

---

## The Golden Rules of Test Code

These rules apply to **every** test — unit and integration alike.

### 1. AAA Structure (Arrange-Act-Assert)

Every test method must follow the AAA pattern with explicit section comments. This is non-negotiable because it makes tests scannable — any developer can look at a test and instantly identify what's being set up, what's being exercised, and what's being verified.

```java
@Test
void should_calculate_total_with_discount() {
    // Arrange
    Order order = anOrder()
            .withItems(List.of(
                    new OrderItem("product-1", 2, new BigDecimal("50.00")),
                    new OrderItem("product-2", 1, new BigDecimal("30.00"))))
            .build();
    BigDecimal discountPercentage = new BigDecimal("10");

    // Act
    BigDecimal total = order.calculateTotalWithDiscount(discountPercentage);

    // Assert
    assertThat(total).isEqualByComparingTo(new BigDecimal("117.00"));
}
```

Keep each section focused: Arrange prepares inputs and mocks, Act calls the SUT (usually one line), Assert verifies the outcome. If Act requires multiple lines, the method under test may be doing too much.

### 2. AssertJ Exclusively

Write all assertions using AssertJ. It produces readable failure messages and supports fluent chaining that makes complex assertions easy to understand:

```java
// Good: AssertJ - failure message is descriptive and clear
assertThat(orders)
        .hasSize(3)
        .extracting(Order::getStatus)
        .containsExactly(OrderStatus.PENDING, OrderStatus.CONFIRMED, OrderStatus.SHIPPED);

assertThat(result.getTotal()).isEqualByComparingTo(new BigDecimal("250.00"));

assertThat(response.getErrors())
        .extracting(Error::getCode)
        .containsExactlyInAnyOrder("INVALID_AMOUNT", "MISSING_CURRENCY");

// Bad: JUnit assertions - failure message is generic ("expected 3 but was 2")
assertEquals(3, orders.size());
assertTrue(orders.stream().allMatch(o -> o.getStatus() == OrderStatus.PENDING));
```

For reactive types, use `StepVerifier` from Project Reactor for stream assertions, combined with AssertJ inside the verification steps:

```java
StepVerifier.create(useCase.findActiveOrders())
        .assertNext(order -> assertThat(order.getStatus()).isEqualTo(OrderStatus.ACTIVE))
        .expectNextCount(2)
        .verifyComplete();
```

### 3. Happy Path AND Sad Path — Always Both

Every test class must cover both successful execution and expected failure scenarios. The sad path tests are where bugs hide — they validate that error handling works correctly and that the system fails gracefully.

**Happy path**: the normal successful flow.

**Sad path**: expected error scenarios — invalid input, entity not found, downstream service unavailable, business rule violation.

```java
@Test
void should_create_order_successfully() {
    // Arrange
    when(orderRepository.save(any())).thenReturn(Mono.just(savedOrder));

    // Act
    Mono<Order> result = useCase.execute(validCommand);

    // Assert
    StepVerifier.create(result)
            .assertNext(order -> {
                assertThat(order.getId()).isNotNull();
                assertThat(order.getStatus()).isEqualTo(OrderStatus.PENDING);
            })
            .verifyComplete();
}

@Test
void should_throw_exception_when_customer_not_found() {
    // Arrange
    when(customerRepository.findById("invalid-id")).thenReturn(Mono.empty());

    // Act
    Mono<Order> result = useCase.execute(commandWithInvalidCustomer);

    // Assert
    StepVerifier.create(result)
            .expectErrorSatisfies(error -> {
                assertThat(error).isInstanceOf(CustomerNotFoundException.class);
                assertThat(error.getMessage()).contains("invalid-id");
            })
            .verify();
}
```

For blocking code, use `assertThatThrownBy` from AssertJ:

```java
@Test
void should_throw_when_amount_is_negative() {
    // Arrange
    BigDecimal negativeAmount = new BigDecimal("-10.00");

    // Act & Assert
    assertThatThrownBy(() -> new Money(negativeAmount, Currency.USD))
            .isInstanceOf(InvalidAmountException.class)
            .hasMessageContaining("Amount must be positive");
}
```

### 4. Test Naming Convention

Use descriptive method names that read as specifications: `should_<expected_behavior>_when_<condition>`. This convention turns the test class into living documentation:

```java
should_create_order_when_all_items_are_available()
should_reject_order_when_stock_insufficient()
should_apply_discount_when_customer_is_premium()
should_throw_not_found_when_order_does_not_exist()
```

---

## Action: Generate

When the user asks to generate tests for a class:

### Step 1: Identify the layer

Read the class and determine which architectural layer it belongs to — this dictates the test type:

| Layer | Test type | Key characteristics |
|-------|-----------|-------------------|
| `domain/model` entities, value objects, enums | Unit | No mocks needed, test pure logic |
| `domain/model` ports | Not directly tested | Tested through their implementations |
| `application/use-cases` | Unit | Mock ports, verify orchestration |
| `driven-adapters/postgres` | Integration | Testcontainers PostgreSQL |
| `driven-adapters/mongo` | Integration | Testcontainers MongoDB |
| `driven-adapters/rabbit-producer` | Integration | Testcontainers RabbitMQ |
| `entry-points/rest-api` | Integration | @WebFluxTest + WebTestClient |
| `entry-points/rabbit-consumer` | Integration | Testcontainers RabbitMQ |

### Step 2: Identify test scenarios

Analyze the production code and list all scenarios before writing any test:

1. **Happy paths**: each public method's normal successful flow
2. **Sad paths**: each expected error case — null inputs, not-found, business rule violations, downstream failures
3. **Edge cases**: empty collections, boundary values, concurrent scenarios (if applicable)

Present the scenario list to the user for confirmation before generating the test code.

### Step 3: Generate the test class

Write the test class following all rules in this skill. Place it in the matching `src/test/java` directory, mirroring the production class package structure.

### Step 4: Verify dependencies

Check if `pom.xml` includes the required test dependencies. If any are missing, tell the user what to add:

```xml
<!-- Always required -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.projectreactor</groupId>
    <artifactId>reactor-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- For integration tests with real databases -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
<!-- Add the specific Testcontainers module: postgresql, mongodb, rabbitmq -->

<!-- For external API simulation -->
<dependency>
    <groupId>org.wiremock</groupId>
    <artifactId>wiremock-standalone</artifactId>
    <scope>test</scope>
</dependency>
```

---

## Action: Review

When reviewing existing tests, check against this checklist in order:

1. **Layer mismatch**: Is a unit test using Spring context? Is an integration test mocking the thing it should be testing against real infra?
2. **Missing sad paths**: Does the test class only cover happy paths? Every public method should have at least one error scenario test.
3. **Assertion quality**: Are assertions using JUnit (`assertEquals`, `assertTrue`) instead of AssertJ? Are assertions too vague (`assertNotNull` when the value should be verified)?
4. **AAA violations**: Is the test structure clear? Are Arrange/Act/Assert sections distinct?
5. **Mock abuse**: Are tests mocking classes they don't own? Mocking concrete classes? Mocking too many dependencies?
6. **State leakage**: Can tests affect each other? Is there proper cleanup between tests?
7. **Naming**: Do test names describe behavior, or are they generic (`test1`, `testCreate`)?
8. **H2/Fongo usage**: Are in-memory databases being used where Testcontainers should be?

Present findings with specific file:line references and concrete fixes.

---

## Action: Refactor

1. Read the existing tests and identify violations from the review checklist above
2. Propose changes grouped by priority: correctness issues first, then structure, then style
3. Apply changes after user confirmation

Common refactorings:
- Replace `@SpringBootTest` with `@ExtendWith(MockitoExtension.class)` in use case tests
- Replace H2 with Testcontainers in adapter tests
- Convert JUnit assertions to AssertJ
- Add missing sad path scenarios
- Extract repeated setup into builders or `@BeforeEach` methods
- Add AAA section comments

---

## Action: Strategy

When the user asks about testing strategy, analyze their project structure and recommend:

1. **What to test at each layer** (mapping from architecture to test types above)
2. **What NOT to test**: trivial getters/setters, framework-generated code (Spring Data repository interfaces), simple DTO mappings with no logic
3. **Priority order**: Domain logic first (highest value, cheapest to test), then use cases, then adapters, then controllers
4. **Test pyramid shape**: Many unit tests (fast, cheap), fewer integration tests (slower, more setup), minimal end-to-end tests (if any)
