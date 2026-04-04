# Resilience Patterns Reference

## Table of Contents
1. [Circuit Breaker](#circuit-breaker)
2. [Dead Letter Queue (DLQ)](#dead-letter-queue)
3. [Idempotent Consumer](#idempotent-consumer)
4. [API Gateway](#api-gateway)
5. [Sidecar Pattern](#sidecar-pattern)
6. [Bulkhead](#bulkhead)
7. [Retry with Backoff](#retry-with-backoff)
8. [Timeout Pattern](#timeout-pattern)
9. [Service Discovery](#service-discovery)
10. [Backpressure](#backpressure)
11. [Graceful Degradation](#graceful-degradation)

---

## Circuit Breaker

Prevents a failing downstream service from cascading failures upstream. The circuit has three states:

```
         success            failure threshold
  ┌─── CLOSED ────────────────► OPEN ───┐
  │    (normal)                (fail fast)│
  │                                      │
  │         ◄── HALF-OPEN ◄─────────────┘
  │             (test with                timeout expires
  │              limited traffic)
  │                │
  │    success     │ failure
  └────────────────┘──────────► OPEN
```

### State Behavior

| State | What Happens | Transitions To |
|-------|-------------|---------------|
| **CLOSED** | Requests pass through normally. Failures are counted. | OPEN (when failure count/rate exceeds threshold) |
| **OPEN** | Requests fail immediately without calling the downstream. Returns fallback response. | HALF-OPEN (after a wait duration) |
| **HALF-OPEN** | A limited number of test requests are sent. | CLOSED (if tests succeed) or OPEN (if tests fail) |

### Configuration Parameters

| Parameter | Description | Typical Value |
|-----------|-------------|---------------|
| `failureRateThreshold` | % of failures to trip the circuit | 50% |
| `slowCallRateThreshold` | % of slow calls to trip | 80% |
| `slowCallDurationThreshold` | What counts as "slow" | 2 seconds |
| `waitDurationInOpenState` | How long before testing again | 30 seconds |
| `permittedCallsInHalfOpen` | Test calls in half-open state | 3 |
| `slidingWindowSize` | Number of calls to evaluate | 10 |

### Fallback Strategies

When the circuit is open, the service should degrade gracefully:

| Strategy | When to Use | Example |
|----------|-------------|---------|
| **Cached response** | Data doesn't change frequently | Return last known product catalog |
| **Default value** | Acceptable approximation exists | Return default shipping estimate |
| **Empty result** | Feature is non-critical | Return empty recommendations list |
| **Queued for retry** | Operation can be deferred | Queue the request, process when circuit closes |
| **Error with context** | Consumer needs to know | Return 503 with retry-after header |

---

## Dead Letter Queue

When a message cannot be processed after multiple retry attempts, it moves to a DLQ instead of blocking the main queue.

### Flow

```
Main Queue ──► Consumer ──► Process
                  │
                  │ (failure, retry 1)
                  ▼
              Consumer ──► Process
                  │
                  │ (failure, retry 2)
                  ▼
              Consumer ──► Process
                  │
                  │ (failure, retry 3 — max reached)
                  ▼
            Dead Letter Queue ──► Alert ──► Manual Review / Auto-Recovery
```

### DLQ Design

| Aspect | Recommendation |
|--------|---------------|
| **Naming** | `<original-queue>.dlq` (e.g., `orders.order-placed.dlq`) |
| **Retention** | Longer than main queue (14-30 days) |
| **Monitoring** | Alert when DLQ depth > 0 |
| **Metadata** | Include original queue, failure reason, retry count, timestamp |
| **Reprocessing** | Provide a mechanism to replay DLQ messages back to the main queue |

### Common Causes of DLQ Messages

| Cause | Fix |
|-------|-----|
| Schema mismatch | Deploy consumer with updated schema |
| Missing required data | Fix producer or add default handling |
| Downstream service permanently down | Wait for recovery, then replay |
| Bug in consumer logic | Fix bug, redeploy, replay |
| Poison message (malformed) | Investigate and discard if irrecoverable |

---

## Idempotent Consumer

Ensures that processing the same message multiple times produces the same result as processing it once. This is essential in systems with at-least-once delivery.

### Implementation Strategies

#### 1. Idempotency Key Store

```sql
CREATE TABLE processed_events (
    event_id    UUID PRIMARY KEY,
    processed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    result      JSONB
);

-- Before processing:
-- 1. Check if event_id exists in processed_events
-- 2. If yes → skip (return cached result if needed)
-- 3. If no → process + INSERT into processed_events (same transaction)
```

#### 2. Natural Idempotency

Design operations to be inherently idempotent:

| Idempotent | Not Idempotent | Fix |
|-----------|---------------|-----|
| `SET balance = 100` | `SET balance = balance + 50` | Use absolute values or check preconditions |
| `INSERT ... ON CONFLICT DO NOTHING` | `INSERT ...` (fails on duplicate) | Use upsert |
| `DELETE WHERE id = X` | `DELETE WHERE created_at < X` (if new rows added) | Use specific identifiers |

#### 3. Optimistic Locking

```sql
UPDATE orders 
SET status = 'CONFIRMED', version = version + 1
WHERE id = :orderId AND version = :expectedVersion;

-- If 0 rows updated → concurrent modification → skip or retry
```

### Cleanup Strategy

The idempotency store grows over time. Implement cleanup:

- **TTL-based**: Delete entries older than the maximum redelivery window (e.g., 7 days)
- **Sliding window**: Keep only the last N hours of event IDs in memory/cache
- **Compaction**: Move old entries to cold storage for audit, remove from hot path

---

## API Gateway

Single entry point for all client requests. Handles cross-cutting concerns before routing to microservices.

### Responsibilities

| Concern | What the Gateway Does |
|---------|----------------------|
| **Routing** | Route requests to the appropriate microservice based on path/headers |
| **Authentication** | Validate JWT/OAuth tokens before forwarding |
| **Rate Limiting** | Protect services from traffic spikes (per-user, per-IP, per-endpoint) |
| **TLS Termination** | Handle HTTPS at the edge, forward HTTP internally |
| **Response Aggregation** | Combine responses from multiple services into one API call |
| **Request Transformation** | Translate external API format to internal service format |
| **Caching** | Cache read-heavy responses at the edge |
| **Circuit Breaking** | Stop forwarding to failing services |
| **CORS** | Handle cross-origin policies centrally |
| **Logging/Tracing** | Generate correlation IDs, start distributed traces |

### Gateway vs. Service Mesh

| Feature | API Gateway | Service Mesh (Istio/Linkerd) |
|---------|------------|------------------------------|
| **Traffic type** | North-south (external → internal) | East-west (service → service) |
| **Scope** | Edge of the system | Between all services |
| **Protocol** | HTTP/REST/GraphQL | Any (HTTP, gRPC, TCP) |
| **Managed by** | API/platform team | Infrastructure/DevOps team |

They are complementary, not competing. Use both when the system grows.

---

## Sidecar Pattern

Deploy auxiliary capabilities alongside a service without modifying its code. The sidecar runs as a separate process/container in the same pod/host.

### Use Cases

| Capability | Sidecar Handles | Service Handles |
|-----------|----------------|----------------|
| **mTLS** | Encrypt/decrypt inter-service traffic | Business logic only |
| **Logging** | Collect and forward logs to aggregator | Write to stdout |
| **Config** | Fetch and inject configuration | Read from environment |
| **Health reporting** | Report metrics to monitoring system | Expose health endpoint |
| **Proxy** | Handle retries, circuit breaking, load balancing | Make simple HTTP calls |

### When Not to Use

- **Small deployments**: Sidecar adds overhead — not worth it for <5 services
- **Latency-critical paths**: Sidecar adds network hop latency (usually 1-2ms)
- **Tight resource constraints**: Each sidecar consumes memory and CPU

---

## Bulkhead

Isolates resource pools so that a failure or overload in one area doesn't starve others.

### Types

| Type | How It Works | Example |
|------|-------------|---------|
| **Thread pool bulkhead** | Separate thread pools per downstream | 10 threads for Orders, 10 for Payments — if Orders hangs, Payment threads are unaffected |
| **Semaphore bulkhead** | Limit concurrent calls per downstream | Max 20 concurrent calls to Inventory service |
| **Process bulkhead** | Separate processes/containers per workload | Critical vs. non-critical consumers on different pods |

### Sizing Guidelines

- Set the bulkhead size based on the downstream's capacity, not the upstream's demand
- Monitor rejection rates — frequent rejections mean the bulkhead is too small or the downstream needs scaling
- Combine with circuit breaker: bulkhead limits concurrency, circuit breaker stops all calls on failure

---

## Retry with Backoff

When a transient failure occurs, retry the operation with increasing delays to avoid overwhelming the failing service.

### Backoff Strategies

| Strategy | Formula | Best For |
|----------|---------|----------|
| **Fixed** | Wait `d` every time | Simple cases, predictable recovery |
| **Exponential** | Wait `d * 2^attempt` | Most distributed systems (default choice) |
| **Exponential + jitter** | Wait `random(0, d * 2^attempt)` | High-concurrency systems (prevents thundering herd) |
| **Decorrelated jitter** | Wait `random(baseDelay, previousDelay * 3)` | AWS recommended — best distribution |

### Configuration

| Parameter | Typical Value |
|-----------|---------------|
| Max retries | 3-5 |
| Initial delay | 100ms - 1s |
| Max delay (cap) | 30s - 60s |
| Jitter | Always enabled |
| Retryable exceptions | Connection timeout, 503, 429 |
| Non-retryable | 400, 401, 403, 404 |

Always use exponential backoff with jitter as the default strategy.

---

## Timeout Pattern

Set explicit timeouts on every external call to prevent thread starvation when a dependency is slow.

### Timeout Hierarchy

```
Client ──► API Gateway (timeout: 30s)
              └──► Service A (timeout: 5s per downstream call)
                      ├──► Database (timeout: 2s)
                      ├──► Service B (timeout: 3s)
                      └──► Message Broker (timeout: 1s)
```

**Rule**: Each layer's timeout must be shorter than its caller's timeout, leaving room for retries and processing. If Service A has a 5s timeout calling Service B, and Service B has a 3s timeout calling the DB, the outer 5s can accommodate one retry.

---

## Service Discovery

How services find and communicate with each other in a dynamic environment where instances scale up/down.

| Approach | How It Works | Best For |
|----------|-------------|----------|
| **DNS-based** | Services register DNS entries; clients resolve hostname | Kubernetes (built-in), simple setups |
| **Registry-based** | Services register with a central registry (Eureka, Consul) | Spring Cloud, VM-based deployments |
| **Service Mesh** | Sidecar proxies handle routing transparently | Large-scale, polyglot environments |
| **Load Balancer** | Central LB routes to healthy instances | Simple architectures, external traffic |

In Kubernetes, use the built-in DNS service discovery (`<service>.<namespace>.svc.cluster.local`). For Spring Boot on VMs, Eureka or Consul are common choices.

---

## Backpressure

When a consumer cannot keep up with the producer's rate, backpressure prevents the system from collapsing under unprocessed messages.

### Strategies

| Strategy | How | Trade-off |
|----------|-----|-----------|
| **Bounded queue** | Set max queue size; reject when full | Producer gets errors, may need retry logic |
| **Rate limiting producer** | Throttle publish rate based on consumer lag | Reduces throughput globally |
| **Consumer scaling** | Auto-scale consumers based on queue depth | Costs more, has spin-up latency |
| **Load shedding** | Drop low-priority messages when overloaded | Data loss for non-critical events |
| **Reactive streams** | Consumer signals demand to producer (Reactor, RxJava) | Requires reactive stack end-to-end |

### Monitoring Metrics

| Metric | Alert Threshold |
|--------|----------------|
| Queue depth | Growing for > 5 minutes |
| Consumer lag | > 10,000 messages behind |
| Processing rate | Declining while publish rate is stable |
| Rejected messages | Any rejections in bounded queue |

---

## Graceful Degradation

When a non-critical service is down, the system should continue operating with reduced functionality rather than failing entirely.

### Degradation Hierarchy

```
Level 0: Full functionality (all services healthy)
Level 1: Reduced features (non-critical service down — hide affected UI, skip optional step)
Level 2: Read-only mode (write service down — serve cached data)
Level 3: Essential only (core service degraded — basic operations only)
Level 4: Maintenance mode (critical failure — inform users, queue requests)
```

### Implementation Patterns

| Pattern | How | Example |
|---------|-----|---------|
| **Feature flags** | Disable features that depend on failing service | Hide "recommendations" when recommendation service is down |
| **Cached fallback** | Return stale cached data when source is unavailable | Show last known inventory count |
| **Default values** | Return safe defaults when data is unavailable | Show "Shipping: 5-7 business days" when shipping calculator is down |
| **Queue and retry** | Accept the request now, process when service recovers | Accept order, process payment later |

The key is to decide ahead of time: for each service dependency, what does the system do when it's unavailable? Document this in the architecture decision records.
