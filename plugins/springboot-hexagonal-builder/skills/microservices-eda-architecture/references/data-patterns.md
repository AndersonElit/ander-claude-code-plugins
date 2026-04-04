# Data Consistency Patterns Reference

## Table of Contents
1. [Saga Pattern](#saga-pattern)
2. [CQRS](#cqrs)
3. [Event Sourcing](#event-sourcing)
4. [Transactional Outbox](#transactional-outbox)
5. [Claim Check](#claim-check)
6. [Pattern Combinations](#pattern-combinations)

---

## Saga Pattern

Manages distributed transactions as a sequence of local transactions, each with a compensating action for rollback.

### Orchestrated Saga

A central orchestrator (Saga Execution Coordinator) directs the flow:

```
┌──────────────────────────────────────────────────────────────┐
│                    Saga Orchestrator                          │
│                                                              │
│  1. CreateOrder ──► OrderService.createOrder()               │
│  2. ReserveStock ──► InventoryService.reserve()              │
│  3. ProcessPayment ──► PaymentService.charge()               │
│  4. ConfirmOrder ──► OrderService.confirm()                  │
│                                                              │
│  Compensation (if step 3 fails):                             │
│  C2. ReleaseStock ──► InventoryService.release()             │
│  C1. CancelOrder ──► OrderService.cancel()                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Implementation guidance:**
- The orchestrator maintains saga state (STARTED, STEP_2_PENDING, COMPENSATING, etc.)
- Each step is idempotent — the orchestrator may retry on timeout
- Store saga state in a database table for crash recovery
- Use a state machine library (e.g., Spring Statemachine) or implement a simple one

### Choreographed Saga

Each service listens for events and reacts, publishing new events for the next step:

```
OrderService          InventoryService       PaymentService
     │                      │                      │
     │─── OrderPlaced ────►│                      │
     │                      │─── StockReserved ──►│
     │                      │                      │─── PaymentCompleted ───►
     │◄── OrderConfirmed ──│◄─────────────────────│
     │                      │                      │
     │  (if PaymentFailed): │                      │
     │                      │◄── PaymentFailed ───│
     │◄── StockReleased ──│                      │
     │─── OrderCancelled   │                      │
```

**Implementation guidance:**
- Each service owns its compensation logic
- The "saga" is implicit — no central coordinator
- Harder to reason about as complexity grows (>3 services)
- Use correlation IDs to track the full saga flow

### Saga Design Checklist

- [ ] Every step has a defined compensating action
- [ ] Compensating actions are idempotent
- [ ] Saga state is persisted (survives crashes)
- [ ] Timeouts are defined for each step
- [ ] There is a maximum retry count before marking the saga as failed
- [ ] Failed sagas trigger alerts for manual review

---

## CQRS

Separates the write model (Commands) from the read model (Queries), allowing each to be optimized independently.

### Architecture

```
                    ┌─────────────────┐
                    │   API Gateway    │
                    └────────┬────────┘
                   ┌─────────┴─────────┐
                   ▼                   ▼
          ┌────────────────┐  ┌────────────────┐
          │  Command Side  │  │  Query Side    │
          │                │  │                │
          │ Validates      │  │ Optimized      │
          │ Applies rules  │  │ read models    │
          │ Writes to      │  │ Denormalized   │
          │ source of truth│  │ views          │
          └───────┬────────┘  └───────▲────────┘
                  │                   │
                  │    Domain Events  │
                  └──────────────────►│
                                      │
          ┌───────────────┐  ┌───────┴────────┐
          │ Write Database │  │ Read Database  │
          │ (normalized)   │  │ (denormalized) │
          └───────────────┘  └────────────────┘
```

### When CQRS Adds Value

| Scenario | Why CQRS Helps |
|----------|---------------|
| Read/write ratio > 10:1 | Scale reads independently |
| Complex domain logic on writes | Keep write model clean, project simpler read models |
| Multiple read representations | Different views for UI, reports, search |
| Event-driven system | Natural fit — events synchronize the read model |

### When CQRS Is Overkill

- Simple CRUD with identical read/write models
- Low traffic where a single model handles both fine
- Small team — the added infrastructure complexity isn't justified

### Read Model Synchronization

The read model is updated asynchronously via domain events. This means:

- **Stale reads are expected** — the read model may lag behind by milliseconds to seconds
- **Rebuild capability** — you must be able to rebuild the entire read model from events
- **Monitoring** — track the lag between write and read models; alert if it exceeds SLA

```
Event consumed → Projection handler → Update read DB → Acknowledge event
```

If the projection handler fails, the event stays in the queue and is retried. The handler must be idempotent.

---

## Event Sourcing

Instead of storing the current state of an entity, store the complete sequence of events that produced that state.

### How It Works

```
Traditional:    Order { status: "CONFIRMED", total: 59.98, items: [...] }

Event Sourced:  
  1. OrderCreated    { orderId, customerId, items, total: 59.98 }
  2. PaymentReceived { orderId, paymentId, amount: 59.98 }
  3. OrderConfirmed  { orderId, confirmedAt }
  
  Current state = replay(event1, event2, event3)
```

### Event Store Design

```sql
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY,
    aggregate_type  VARCHAR(100) NOT NULL,
    aggregate_id    UUID NOT NULL,
    sequence_number BIGINT NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    event_data      JSONB NOT NULL,
    metadata        JSONB,
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE (aggregate_id, sequence_number)
);

CREATE INDEX idx_event_store_aggregate 
    ON event_store (aggregate_id, sequence_number);
```

### Snapshots

Replaying thousands of events for every read is expensive. Snapshots solve this:

1. Every N events (e.g., 100), save the current state as a snapshot
2. To rebuild state: load latest snapshot + replay events after it
3. Snapshots are an optimization — the event log remains the source of truth

### When to Use Event Sourcing

| Good Fit | Poor Fit |
|----------|----------|
| Audit and compliance requirements | Simple CRUD applications |
| Need to answer "what happened and when?" | Team unfamiliar with the pattern |
| Complex domain with many state transitions | Data with frequent deletes (GDPR compliance adds complexity) |
| Event replay for debugging or new projections | High-frequency updates on same entity (snapshot overhead) |

### GDPR and Event Sourcing

Events are immutable by design, but GDPR requires the ability to delete personal data. Strategies:

- **Crypto-shredding**: Encrypt PII with a per-user key; to "delete", destroy the key
- **Event transformation**: Replace sensitive fields with tombstone values in a new stream
- **Reference tokens**: Store PII in a separate mutable store, reference it by token in events

---

## Transactional Outbox

Solves the dual-write problem: updating a database AND publishing an event must be atomic. Without this, you risk the database being updated but the event never being sent (or vice versa).

### How It Works

```
┌─────────────── Same Database Transaction ───────────────┐
│                                                          │
│  1. UPDATE orders SET status = 'CONFIRMED'               │
│  2. INSERT INTO outbox (event_type, payload, status)     │
│     VALUES ('OrderConfirmed', '{...}', 'PENDING')        │
│                                                          │
│  COMMIT                                                  │
└──────────────────────────────────────────────────────────┘

┌─────────────── Separate Process (Poller or CDC) ────────┐
│                                                          │
│  3. SELECT * FROM outbox WHERE status = 'PENDING'        │
│  4. Publish to message broker                            │
│  5. UPDATE outbox SET status = 'SENT'                    │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Outbox Table Schema

```sql
CREATE TABLE outbox (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(100) NOT NULL,
    aggregate_id    VARCHAR(100) NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    payload         JSONB NOT NULL,
    status          VARCHAR(20) DEFAULT 'PENDING',
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    published_at    TIMESTAMP WITH TIME ZONE,
    retry_count     INTEGER DEFAULT 0
);

CREATE INDEX idx_outbox_pending ON outbox (status, created_at) 
    WHERE status = 'PENDING';
```

### Relay Strategies

| Strategy | How | Pros | Cons |
|----------|-----|------|------|
| **Polling publisher** | Periodically query the outbox table | Simple, works with any DB | Latency (polling interval), DB load |
| **CDC (Change Data Capture)** | Stream DB changes via Debezium/similar | Near real-time, no polling | Infrastructure complexity |
| **Transaction log tailing** | Read the DB's WAL/binlog directly | Lowest latency | DB-specific, complex setup |

For most projects, start with polling (every 1-5 seconds). Move to CDC when latency or throughput demands it.

---

## Claim Check

When an event payload exceeds the broker's message size limit or is inefficient to transmit, store the large data externally and pass only a reference.

### How It Works

```
Producer:
  1. Store large payload in S3/Blob Storage → get reference key
  2. Publish event with { "claimCheckRef": "s3://bucket/key" }

Consumer:
  1. Receive event with claim check reference
  2. Fetch full payload from S3/Blob Storage
  3. Process normally
```

### When to Use

| Use Claim Check | Send Directly |
|----------------|---------------|
| Payload > 256KB | Payload < 256KB |
| Binary content (images, PDFs, files) | Structured JSON data |
| Payload needed by only one consumer | Payload needed by all consumers |
| Sensitive data requiring access control | Non-sensitive operational data |

### Implementation Notes

- Set TTL on stored objects to auto-clean after consumers have processed them
- Include metadata in the event (file size, content type, checksum) so consumers can validate before fetching
- Handle the case where the external store is unavailable — retry with backoff

---

## Pattern Combinations

These patterns often work together. Common combinations:

| Combination | Why They Fit Together |
|-------------|----------------------|
| **CQRS + Event Sourcing** | Events naturally feed the read model projections |
| **Saga + Outbox** | Each saga step writes to DB + outbox atomically |
| **Event Sourcing + Saga** | Saga events become part of the event store |
| **CQRS + API Gateway** | Gateway routes commands to write service, queries to read service |
| **Outbox + CDC** | Debezium captures outbox inserts and publishes them |

### Anti-Patterns to Avoid

| Anti-Pattern | Problem | Alternative |
|-------------|---------|-------------|
| **Distributed monolith** | Microservices that must deploy together | Revisit bounded contexts — they're too coupled |
| **Shared database** | Physical coupling defeats service independence | Database-per-service + events for data sharing |
| **Synchronous event handling** | Blocks the producer, reintroduces coupling | Use async messaging; if sync is needed, use direct API calls |
| **God event** | One massive event with every field | Separate events per concern; consumers subscribe to what they need |
| **Event-driven CRUD** | Events for simple create/update with no subscribers | Use direct API calls if nobody cares about the event |
| **Two-phase commit** | Distributed locks across services | Saga pattern with compensating transactions |
