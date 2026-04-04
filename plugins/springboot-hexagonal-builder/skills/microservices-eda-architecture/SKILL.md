---
name: microservices-eda-architecture
description: "Design microservices architectures and Event-Driven Architecture (EDA) systems from business requirements. Use this skill whenever the user needs to decompose a system into microservices, define bounded contexts, identify domain events, design event-driven flows, choose between choreography and orchestration, define event contracts (Avro/JSON Schema), apply distributed patterns (Saga, CQRS, Event Sourcing, Outbox, Circuit Breaker), design messaging topologies, plan data consistency strategies, or architect resilient distributed systems. Also triggers for 'microservices design', 'event-driven architecture', 'domain decomposition', 'bounded contexts', 'event storming', 'saga pattern', 'CQRS', 'event sourcing', 'transactional outbox', 'distributed transactions', 'choreography vs orchestration', 'messaging architecture', 'event contracts', 'dead letter queue', 'circuit breaker pattern', 'API gateway design', 'database per service', 'eventual consistency', 'distributed tracing', 'claim check pattern', 'event versioning', 'consumer-driven contracts', 'arquitectura de microservicios', 'arquitectura orientada a eventos', 'descomposicion de dominios', 'contextos acotados', 'patrones distribuidos', 'consistencia eventual', 'sagas distribuidas'."
---

# Microservices & Event-Driven Architecture Designer

Transform business requirements into well-structured microservices with event-driven communication. This skill guides architectural design decisions — from domain decomposition to pattern selection — ensuring the resulting system is resilient, scalable, and maintainable.

## Workflow Overview

The design flows through six phases. Each phase builds on the previous one, but you can jump to any phase if the user has already completed earlier work.

| Phase | Name | Output |
|-------|------|--------|
| 1 | Domain Decomposition | Bounded context map + service boundaries |
| 2 | Event Identification | Domain events catalog + event flows |
| 3 | Communication Design | Pattern selection (choreography/orchestration) + messaging topology |
| 4 | Data Strategy | Consistency patterns per service interaction |
| 5 | Resilience Design | Fault tolerance patterns + observability plan |
| 6 | Architecture Document | Consolidated architecture decision record |

---

## Phase 1: Domain Decomposition

Before writing any code or choosing any technology, map the business domain. The goal is to find natural service boundaries where each microservice owns a cohesive set of business capabilities and its own data.

### Step 1: Identify Business Capabilities

Ask the user (or analyze existing docs/code) to enumerate what the system **does** — not what it **is**. Group related actions:

| Business Capability | Key Operations | Data Owned |
|---------------------|---------------|------------|
| Order Management | Create, modify, cancel orders | Orders, line items |
| Inventory | Reserve, release, adjust stock | Stock levels, warehouses |
| Payment Processing | Charge, refund, reconcile | Transactions, payment methods |

### Step 2: Draw Bounded Context Boundaries

Apply these heuristics to decide where one service ends and another begins:

- **Language boundary**: If the same word means different things in two areas (e.g., "account" in billing vs. user management), they belong in separate contexts.
- **Change cadence**: Features that evolve together belong together. If orders change weekly but inventory rules change quarterly, they are separate contexts.
- **Team ownership**: If different teams own different areas, respect that — Conway's Law is real.
- **Data ownership**: Each context must own its authoritative data. If two contexts need the same data, one owns it and the other receives copies via events.
- **Transactional boundary**: Operations that must be strictly consistent (ACID) should live in the same service. Operations that tolerate eventual consistency can cross service boundaries.

### Step 3: Define Context Relationships

Map how bounded contexts interact using these relationship types:

| Relationship | Direction | Meaning | Example |
|-------------|-----------|---------|---------|
| **Upstream/Downstream** | U → D | Upstream publishes, downstream consumes | Orders → Notifications |
| **Shared Kernel** | Bidirectional | Two contexts share a small common model | Shared domain types |
| **Anti-Corruption Layer** | D → U | Downstream translates upstream's model | Legacy integration |
| **Conformist** | D → U | Downstream adopts upstream's model as-is | Using a third-party API |
| **Open Host Service** | U → D | Upstream exposes a well-defined protocol | Public API |

### Step 4: Validate with Database-per-Service

Each microservice gets its own database — no sharing. This is non-negotiable because a shared database creates physical coupling that defeats the purpose of microservices. Validate your decomposition by checking:

- Can each service operate with only its own data store?
- If a service needs data from another, can it get it asynchronously via events?
- Are there "god tables" that multiple services need to write to? If so, the boundaries need adjustment.

**Output**: A bounded context map showing services, their owned data, and their relationships. Generate this as a Mermaid diagram.

---

## Phase 2: Event Identification

Identify every significant state change in the system that other services might care about.

### Step 1: Catalog Domain Events

A domain event represents something that **happened** — always named in past tense:

| Bounded Context | Event Name | Trigger | Key Payload Fields |
|----------------|------------|---------|-------------------|
| Orders | `OrderPlaced` | Customer submits order | orderId, customerId, items[], totalAmount |
| Orders | `OrderCancelled` | Customer or system cancels | orderId, reason, cancelledBy |
| Payments | `PaymentCompleted` | Payment processor confirms | paymentId, orderId, amount, method |
| Inventory | `StockReserved` | Inventory locks units | reservationId, orderId, items[] |

### Step 2: Classify Event Types

Not all events are equal. Classify them to choose the right handling strategy:

| Type | Description | Example | Typical Pattern |
|------|-------------|---------|----------------|
| **Domain Event** | Business state change | `OrderPlaced` | Publish to topic |
| **Integration Event** | Cross-boundary notification | `PaymentCompleted` | Event bus / broker |
| **Command Event** | Request for action | `ProcessRefund` | Direct queue |
| **Notification Event** | Informational, no action required | `DailyReportGenerated` | Fan-out |

### Step 3: Define Event Contracts

Events are **public APIs** — treat them with the same rigor. For each event, define a contract.

Read `references/event-contracts.md` for JSON Schema and Avro templates, versioning strategy, and envelope structure.

**Key rules for event contracts:**
- Include a standard envelope: `eventId`, `eventType`, `version`, `timestamp`, `correlationId`, `source`
- The payload should be self-contained — a consumer should not need to call back to the producer to understand the event
- Never include sensitive data (PII, credentials) in events without encryption
- Use semantic versioning for event schemas

---

## Phase 3: Communication Design

Choose how services coordinate based on the nature of each interaction.

### Choreography vs. Orchestration

| Criteria | Choreography | Orchestration |
|----------|-------------|---------------|
| **Coupling** | Minimal — services react independently | Higher — orchestrator knows the flow |
| **Visibility** | Hard to trace end-to-end flow | Easy — orchestrator has the full picture |
| **Complexity** | Grows with number of participants | Centralized in the orchestrator |
| **Failure handling** | Each service handles its own | Orchestrator manages compensation |
| **Best for** | Simple event chains (2-3 services) | Complex business processes, Sagas with many steps |

**Decision guide:**
- If the flow involves **3 or fewer services** and no complex rollback logic → **Choreography**
- If the flow involves **4+ services**, has conditional branching, or needs coordinated rollback → **Orchestration**
- If the flow is a mix → Use **choreography** for the happy path and **orchestration** for the compensation/error path

### Messaging Topology

Design the topic/queue structure:

| Pattern | When to Use | Example |
|---------|-------------|---------|
| **Topic per event type** | Default choice — one topic per domain event | `orders.order-placed`, `payments.payment-completed` |
| **Topic per aggregate** | When consumers need all events for an entity | `orders.order-events` (placed, modified, cancelled) |
| **Command queue** | Point-to-point commands | `inventory.reserve-stock` |
| **Reply queue** | Request-reply over messaging | `payments.process-payment.reply` |

**Naming convention**: `<bounded-context>.<event-or-command-name>` in kebab-case.

---

## Phase 4: Data Strategy

Microservices cannot use distributed ACID transactions. Choose the right consistency pattern for each cross-service interaction.

Read `references/data-patterns.md` for detailed implementation guidance on each pattern.

### Pattern Selection Matrix

| Scenario | Recommended Pattern | Complexity | When to Avoid |
|----------|-------------------|------------|---------------|
| Multi-service transaction with rollback | **Saga** (orchestrated) | High | Simple CRUD, < 3 services |
| Multi-service transaction, loosely coupled | **Saga** (choreographed) | Medium | Complex branching logic |
| High read load, separate read/write models | **CQRS** | Medium | When read/write models are identical |
| Full audit trail, temporal queries, replay | **Event Sourcing** | Very High | Simple CRUD with no audit needs |
| Atomic DB write + event publish | **Transactional Outbox** | Medium | In-memory/stateless services |
| Large event payloads | **Claim Check** | Low | Small payloads (< 256KB) |

### Complexity vs. Benefit Decision

This is critical: do not apply a complex pattern where a simpler one suffices. The decision should be driven by business requirements, not technical enthusiasm.

- **Start simple**: If a standard REST call with retry logic works, use it.
- **Add Sagas** only when you genuinely need distributed rollback.
- **Add CQRS** only when read and write workloads have fundamentally different scaling or modeling needs.
- **Add Event Sourcing** only when the business explicitly needs audit trails, temporal queries, or event replay capabilities.

---

## Phase 5: Resilience Design

A distributed system is only as strong as its weakest failure-handling strategy.

Read `references/resilience-patterns.md` for implementation details on each pattern.

### Resilience Checklist

For each service-to-service interaction, verify these protections are in place:

| Protection | Pattern | Purpose |
|-----------|---------|---------|
| **Downstream failure** | Circuit Breaker | Stop cascading failures by failing fast |
| **Message processing failure** | Dead Letter Queue (DLQ) | Isolate poison messages without blocking the pipeline |
| **Duplicate delivery** | Idempotent Consumer | Process the same message multiple times safely |
| **Slow dependency** | Timeout + Retry with backoff | Avoid thread starvation; exponential backoff prevents thundering herd |
| **Overload** | Rate Limiting / Bulkhead | Isolate resource pools so one slow consumer doesn't starve others |
| **Single entry point** | API Gateway | Centralize auth, rate limiting, routing, TLS termination |
| **Cross-cutting concerns** | Sidecar | Add observability, security, or config without modifying service code |

### Idempotency Strategy

In event-driven systems with at-least-once delivery, every consumer must handle duplicate messages. Common strategies:

| Strategy | How It Works | Best For |
|----------|-------------|----------|
| **Idempotency key** | Store processed event IDs; skip if already seen | Commands, payment processing |
| **Natural idempotency** | Operation is inherently safe to repeat (e.g., SET vs. INCREMENT) | Status updates, overwrites |
| **Optimistic locking** | Version field on entity; reject stale updates | Concurrent writes |
| **Deduplication window** | Track event IDs for a time window, then expire | High-throughput event streams |

### Observability Plan

Distributed systems are impossible to debug without proper observability. Every architecture must include:

| Pillar | What | Implementation |
|--------|------|----------------|
| **Distributed Tracing** | Trace a request across services and queues | Propagate `traceId` and `spanId` in headers and event envelopes |
| **Correlation ID** | Link all events/logs belonging to one business transaction | Generate at entry point, propagate everywhere |
| **Structured Logging** | Machine-parseable logs with consistent fields | JSON logs with `correlationId`, `serviceId`, `eventType` |
| **Health Checks** | Service and dependency status | `/actuator/health` with liveness and readiness probes |
| **Metrics** | Latency, throughput, error rates, queue depth | Micrometer/Prometheus with dashboards per service |
| **Alerting** | DLQ depth, circuit breaker state, error rate spikes | Threshold-based alerts on critical metrics |

---

## Phase 6: Architecture Document

Consolidate all decisions into a single architecture document. Save to `docs/design/architecture/MICROSERVICES-EDA-ARCHITECTURE.md`.

### Document Template

```markdown
# Microservices & EDA Architecture — {Project Name}

## 1. Domain Decomposition
### Bounded Context Map
{Mermaid diagram from Phase 1}

### Service Catalog
| Service | Bounded Context | Database | Tech Stack |
|---------|----------------|----------|------------|

## 2. Event Catalog
### Domain Events
{Table from Phase 2}

### Event Flow Diagrams
{Mermaid sequence diagrams showing key flows}

## 3. Communication Architecture
### Pattern Decisions
| Interaction | Pattern | Justification |
|-------------|---------|---------------|

### Messaging Topology
{Topic/queue structure diagram}

## 4. Data Consistency Strategy
| Cross-Service Flow | Pattern | Compensation Strategy |
|-------------------|---------|----------------------|

## 5. Resilience Architecture
### Per-Service Protection Matrix
| Service | Circuit Breaker | DLQ | Idempotency | Rate Limit |
|---------|----------------|-----|-------------|------------|

### Observability Stack
{Tools and configuration}

## 6. Architecture Decision Records (ADRs)
{One ADR per significant decision — why this pattern over alternatives}

## 7. Deployment Topology
{Container/pod layout, service mesh, gateway configuration}

## 8. Evolution Strategy
### Event Versioning Policy
### Consumer-Driven Contract Testing
### Migration Playbooks
```

### Additional Aspects to Document

These are often overlooked but critical for production systems:

- **Service Discovery**: How services find each other (DNS, Consul, Kubernetes Service, Eureka)
- **Configuration Management**: Externalized config (Spring Cloud Config, Vault, ConfigMaps)
- **Secret Management**: How credentials and API keys are managed (Vault, AWS Secrets Manager, sealed secrets)
- **Data Synchronization**: How read models stay in sync in CQRS (lag tolerance, rebuild procedures)
- **Schema Registry**: Central registry for event schemas (Confluent Schema Registry, AWS Glue) to enforce compatibility
- **Backpressure Handling**: What happens when a consumer can't keep up (buffer, drop, throttle producer)
- **Graceful Degradation**: What the system does when a non-critical service is down (fallback responses, cached data, feature flags)
- **Testing Strategy**: Contract tests (Pact), integration tests with test containers, chaos engineering approach
- **Data Migration**: How to handle schema evolution in database-per-service (Flyway/Liquibase per service, blue-green migrations)
- **Event Replay & Recovery**: Procedures for replaying events after a consumer failure or to bootstrap a new service

---

## Cross-Skill Integration

This skill integrates with other skills in the plugin:

| When | Invoke |
|------|--------|
| Need to scaffold a microservice | `/hexagonal-architecture-builder` — generates the service with hexagonal architecture |
| Need to design the relational DB for a service | `/relational-db-schema-builder` — ER diagrams, DDL, data dictionary |
| Need to design NoSQL collections | `/nosql-schema-builder` — document modeling, JSON Schema, indexes |
| Need to document the API contract | `/openapi-doc-builder` — OpenAPI 3.x spec |
| Need C4 diagrams of the architecture | `/c4-architecture` — system context, container, component diagrams |
| Need to define test strategy | `/java-testing-architect` — testing patterns for hexagonal microservices |

After completing the architecture design, offer to invoke these skills for implementation.
