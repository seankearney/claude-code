# Microservices Patterns (microservices.io)

> Shared knowledge module. Referenced by agents that evaluate distributed service designs.

## Decomposition Patterns

| Pattern | Description | Detection Signal |
|---|---|---|
| **Decompose by Business Capability** | Services aligned to business functions (Payments, Billing, Notifications) | Service names reflect business domains, not technical layers |
| **Decompose by Subdomain** (DDD) | Services aligned to bounded contexts | Ubiquitous language within service, anti-corruption layers at boundaries |
| **Strangler Fig** | Incrementally replace legacy with new services | Facade routing traffic to old/new systems, feature flags for migration |
| **Self-Contained System** | Each service includes UI + logic + data | Reduces cross-team coordination, may duplicate UI patterns |

**Anti-pattern: Distributed Monolith** — Services that must be deployed together, share databases, or have synchronous call chains. Worse than a monolith (distributed complexity without independence).

## Communication Patterns

### Synchronous

| Pattern | Use When | Watch For |
|---|---|---|
| **REST (HTTP/JSON)** | Simple CRUD, wide client compatibility | Overfetching, chatty APIs, missing HATEOAS |
| **gRPC** | Internal service-to-service, performance-critical, streaming | Schema evolution (protobuf), HTTP/2 requirement |
| **GraphQL** | Frontend-driven queries, aggregating multiple services | N+1 queries, authorization complexity |

### gRPC Durability Gaps

gRPC is a synchronous RPC protocol — it provides **no delivery guarantees** beyond the lifetime of a single connection. This matters when services treat gRPC like a reliable message bus.

| Gap | Risk | Mitigation |
|---|---|---|
| **No persistence** | If the server is down, the call fails and the payload is lost | Front critical calls with a transactional outbox or durable queue |
| **No built-in retry** | Transport errors require application-level retry logic | Use gRPC retry policy config or Polly; **caller must ensure idempotency** (see `idempotency-patterns.md`) |
| **Stream disconnection** | Long-lived streams silently drop on network blips, load balancer resets, or pod reschedules | Implement keepalive pings (`GRPC_ARG_KEEPALIVE_TIME_MS`), reconnection logic, and stream-resume tokens |
| **No dead-lettering** | Failed calls vanish — no DLQ, no audit trail | Log failed payloads to a fallback store; alert on repeated failures |
| **No back-pressure signaling** | Server overload manifests as timeouts or `RESOURCE_EXHAUSTED`, not flow control | Use gRPC flow control windows, set `MaxConcurrentStreams`, implement client-side throttling |

**When to use gRPC vs. a message queue:**
- **gRPC:** Request needs a synchronous response, low-latency requirement, internal service mesh with retries configured
- **Message queue:** Fire-and-forget commands, work that must survive service restarts, fan-out to multiple consumers, operations requiring guaranteed delivery

**Detection signals:** Critical mutations (payments, state changes) sent over gRPC with no fallback persistence, no retry policy configured, long-lived streams with no keepalive or reconnection handling, gRPC used between services that don't need synchronous responses.

### Asynchronous

| Pattern | Use When | Watch For |
|---|---|---|
| **Message Queue** (point-to-point) | Command-style operations, work distribution | Dead letter queues, poison messages, ordering |
| **Publish/Subscribe** (fan-out) | Event notification to multiple consumers | Consumer lag, event schema evolution |
| **Request/Reply** (async) | Long-running operations with callback | Correlation IDs, timeout handling, reply routing |

### API Gateway / BFF

- **API Gateway:** Single entry point for all clients. Handles routing, auth, rate limiting, request aggregation.
- **Backends for Frontends (BFF):** Separate gateway per client type (web, mobile, IoT). Each BFF tailored to its client's needs.
- **Detection of need:** Clients making 5+ API calls to render a single view — candidate for gateway aggregation.

### API Versioning

| Strategy | Example | Trade-offs |
|---|---|---|
| **URL Path** | `/api/v2/payments` | Simple, visible, but pollutes routing; hard to share middleware across versions |
| **Header** | `Api-Version: 2` | Clean URLs, but invisible in browser/logs; requires client awareness |
| **Media Type** | `Accept: application/vnd.myapp.v2+json` | Most RESTful, but complex content negotiation; tooling support varies |

**Backward Compatibility Rules:**
- **Safe (non-breaking):** Adding new fields, new endpoints, new optional query params
- **Breaking:** Removing/renaming fields, changing field types, altering response structure, changing error codes
- **Rule of thumb:** Additive-only changes never need a version bump

**Detection signals:** No versioning strategy at all, breaking changes shipped without version bump, multiple active versions with no deprecation timeline.

### Messaging Patterns

**Competing Consumers:**
- Multiple instances consume from the same queue for horizontal scaling
- Broker assigns each message to exactly one consumer (visibility timeout / message lock)
- If consumer crashes before ACK, message redelivers → **consumer must be idempotent** (see `idempotency-patterns.md`)
- Detection: Single consumer bottleneck, no visibility timeout configured, messages processed twice after crash

**Message Ordering:**
- **When ordering matters:** Financial ledger entries, state-machine transitions, event sourcing
- **When it doesn't:** Independent notifications, idempotent upserts, commutative operations
- **Partition-based ordering:** Messages with same partition key (e.g., account ID) go to same partition → ordered within that key
- **Alternative:** Design consumers to be idempotent and order-independent — more scalable than enforcing strict ordering
- Detection: Assumes FIFO without configuring it, no partition key strategy, ordering bugs in production

**Dead Letter Queues (DLQ):**
- Messages route to DLQ after: max retry count exceeded, TTL expired, consumer explicitly rejects
- **Monitor DLQ depth** — non-zero depth means messages are failing permanently
- Replay strategy: Fix bug → replay DLQ messages back to main queue (must be idempotent)
- **Poison message isolation:** Messages that crash consumers must be identified and sidelined, not retried forever
- Detection: No DLQ configured, DLQ exists but nobody monitors it, no replay procedure documented

**Back-Pressure / Flow Control:**
- **Prefetch limit:** Cap how many unacknowledged messages a consumer holds (prevents memory exhaustion)
- **Queue depth alerts:** Alert when queue depth exceeds threshold — consumers can't keep up
- **Throttling:** Slow producers when queue is near capacity (reject with 429, or broker-level flow control)
- Without back-pressure: queues grow unbounded → broker runs out of memory/disk → total message loss
- Detection: No prefetch limit, no queue depth monitoring, unbounded in-memory queues

## Data Management Patterns

### Saga Pattern

Manage distributed transactions without 2PC (two-phase commit):

**Choreography** (event-driven):
- Each service listens for events and publishes next event
- Pro: Decoupled, simple for 2-3 step flows
- Con: Hard to track overall progress, circular dependencies risk

**Orchestration** (coordinator):
- Central orchestrator directs each step and handles compensation
- Pro: Clear flow visibility, easier error handling
- Con: Single point of failure, orchestrator can become god object

**Compensation:** Every step must have a compensating action for rollback:
```
CreateOrder → CancelOrder
ReserveInventory → ReleaseInventory
ChargePayment → RefundPayment
```

### Transactional Outbox

```
BEGIN TRANSACTION
  INSERT INTO Orders (...)
  INSERT INTO Outbox (event_type, payload, created_at)
COMMIT

-- Separate polling process:
SELECT * FROM Outbox WHERE published = false
-- Publish to message broker
-- Mark as published
```

**Why:** Guarantees atomicity between state change and event publication. Without this, you get "dual write" problems (DB write succeeds but event publish fails, or vice versa).

## Reliability Patterns

### Circuit Breaker

Three states: **Closed** (normal) → **Open** (failing, fast-fail) → **Half-Open** (testing recovery)

```
Closed: Requests pass through. Track failure count.
  → If failures exceed threshold → Open

Open: All requests fail immediately (no remote call).
  → After timeout → Half-Open

Half-Open: Allow limited requests through.
  → If success → Closed
  → If failure → Open
```

**In .NET:** Use Polly (`AddPolicyHandler`) or .NET 8+ `Microsoft.Extensions.Http.Resilience`.

**Detection of need:** Service calls without any failure handling, retries without backoff, cascading failures.

### Bulkhead

Isolate failures so one failing dependency doesn't exhaust all resources:
- **Thread pool bulkhead:** Separate thread pools per downstream service
- **Semaphore bulkhead:** Limit concurrent calls to a dependency
- **Detection:** All HTTP calls sharing one `HttpClient` with no concurrency limits

### Retry with Exponential Backoff + Jitter

```
delay = min(base * 2^attempt + random_jitter, max_delay)
```

- **Jitter is mandatory** — without it, retries from multiple clients synchronize ("thundering herd")
- **Idempotency required** — retried operations must be safe to repeat
- **Detection:** Retry loops with fixed delay, retries without idempotency checks

### Health Check API

- **Liveness:** Is the process alive? (restart if not)
- **Readiness:** Can the service handle requests? (remove from load balancer if not)
- **Startup:** Has the service finished initializing? (don't check liveness until started)
- **Deep health checks:** Verify downstream dependencies (DB, cache, message broker)
- **Detection:** Services with no `/health` endpoint, or health checks that always return 200

### Timeout Cascading & Deadline Propagation

Timeouts must **shrink** through a call chain — each downstream call must have a shorter timeout than its caller:

```
API Gateway (30s) → Service A (25s) → Service B (20s) → Database (5s)
```

**Deadline Propagation Pattern:**
- Caller passes an **absolute deadline** (e.g., `Deadline: 2024-01-15T10:00:30Z`) instead of a relative timeout
- Each hop calculates remaining budget: `remaining = deadline - now`
- If remaining ≤ 0, fail immediately without making the downstream call (saves wasted work)
- **In .NET:** Pass deadline via `CancellationToken` with `CancellationTokenSource.CancelAfter(remaining)`

**Detection signals:** Same timeout value at every layer, no timeout configured at all, downstream timeout longer than caller's timeout, no deadline/cancellation token propagation.

### Graceful Degradation

When a dependency is unavailable (circuit breaker open, timeout, error), the service should degrade rather than fail entirely.

**Fallback Hierarchy** (try in order):
1. **Cached response** — Return last-known-good data (with staleness indicator)
2. **Default value** — Return a sensible default (e.g., default exchange rate, empty recommendations)
3. **Partial response** — Return what you can, omit the failed section (e.g., product page without reviews)
4. **Informative error** — Clear message explaining what's degraded and expected recovery

**Feature Degradation Matrix:**

| Feature | Required Dependencies | Degraded Behavior When Unavailable |
|---|---|---|
| Payment Processing | Payment Gateway, DB | Queue for retry, return "pending" status |
| Account Balance | DB | Return cached balance with "as of" timestamp |
| Notifications | Email/SMS service | Queue silently, no user-facing error |
| Reporting | Analytics DB | Show "temporarily unavailable" banner |

**Detection signals:** Circuit breaker opens but no fallback defined (returns raw 500/503), service has no cached data strategy, all-or-nothing responses (entire page fails because one widget's API is down), no degradation testing in CI.

## Observability Patterns

### Distributed Tracing

- Every request gets a **correlation ID** (trace ID) propagated across all services
- Each service creates **spans** within the trace
- Enables end-to-end latency analysis and failure diagnosis
- **In .NET:** `System.Diagnostics.Activity`, OpenTelemetry SDK
- **Detection:** Missing `Activity` propagation, no trace context in HTTP headers (`traceparent`)

### Log Aggregation

- Structured logging with correlation IDs in every log entry
- Centralized log storage (ELK, Splunk, CloudWatch)
- **Detection:** `Console.WriteLine` instead of `ILogger`, missing correlation context, unstructured log messages

### Application Metrics

- RED metrics: **R**ate, **E**rrors, **D**uration for every service endpoint
- USE metrics: **U**tilization, **S**aturation, **E**rrors for every resource
- Business metrics: Payments processed, orders completed, etc.
- **Detection:** No metrics instrumentation, no dashboards, no SLO definitions

## Deployment Patterns

| Pattern | Description | Risk Level |
|---|---|---|
| **Blue-Green** | Two identical environments, switch traffic instantly | Low — instant rollback |
| **Canary** | Route small % of traffic to new version, gradually increase | Low — controlled exposure |
| **Rolling** | Replace instances one at a time | Medium — mixed versions during deploy |
| **Feature Flags** | Deploy code disabled, enable per-user/region/% | Low — decouple deploy from release |
| **Sidecar** | Deploy helper process alongside main service (logging, proxy) | Low — no main service changes |

## Testing Patterns

| Pattern | Scope | Purpose |
|---|---|---|
| **Unit Tests** | Single class/function | Verify business logic in isolation |
| **Integration Tests** | Service + real dependencies | Verify data access, external API contracts |
| **Contract Tests** | API consumer ↔ provider | Verify API compatibility without full integration |
| **Component Tests** | Entire service in isolation | Verify service behavior with mocked dependencies |
| **End-to-End Tests** | Full system | Verify critical user journeys (keep minimal) |

**Consumer-Driven Contract Testing:** Consumer defines expected API behavior → Provider verifies it can fulfill the contract. Catches breaking changes before deployment. Tools: Pact.
