# Architecture Patterns

> Shared knowledge module. Referenced by agents that evaluate system design and structure.

## Layered Architecture

Proper dependency direction: **UI → Application → Domain → Infrastructure**

| Violation | Detection Signal | Impact |
|---|---|---|
| Controller → Repository directly | Controller references `DbContext` or `SqlConnection` | Bypasses business rules |
| Domain → Infrastructure | Domain entity imports `System.Data` or `HttpClient` | Core logic coupled to I/O |
| Circular dependencies | Project A references B, B references A | Build issues, tangled logic |
| Layer skipping | UI calls data access, skipping application/domain | Missing validation, audit, business rules |

## Clean Architecture / Hexagonal (Ports & Adapters)

- **Core domain** is isolated from infrastructure — no framework imports in domain entities
- **Ports** = interfaces defined in the domain/application layer (e.g., `IPaymentGateway`)
- **Adapters** = implementations in infrastructure (e.g., `StripePaymentGateway : IPaymentGateway`)
- **Application services** are use case orchestrators — they coordinate domain objects and ports
- **Driving adapters** (primary): Controllers, message handlers, CLI — they call application services
- **Driven adapters** (secondary): Repositories, API clients, message publishers — called by application services via ports

**Detection:** If removing a NuGet package (e.g., `Npgsql`, `AWS.SDK`) requires changing domain classes, the architecture is coupled.

## CQRS (Command Query Responsibility Segregation)

- **Commands** modify state, return void or result (no data queries)
- **Queries** return data, have no side effects
- Separate read/write models when read patterns diverge from write patterns
- **When appropriate:** High-read/low-write ratios, complex domain logic, different scaling needs for reads vs. writes
- **When over-engineering:** Simple CRUD apps, low traffic, small team
- **Event Sourcing** (optional complement): Store state as sequence of events. Enables audit trail, temporal queries, event replay. Adds complexity — only when audit/replay is genuinely needed.

## Event-Driven Architecture

| Pattern | Use When | Watch For |
|---|---|---|
| **Domain Events** | Side effects within a bounded context (e.g., `OrderPlaced` → update inventory) | Events must be handled in same transaction or via outbox |
| **Integration Events** | Cross-service communication (e.g., `PaymentCompleted` → notify shipping) | Must be idempotent, handle out-of-order delivery |
| **Event Sourcing** | Full audit trail needed, temporal queries, event replay | Complexity, eventual consistency, snapshot management |
| **Eventual Consistency** | Distributed systems where strong consistency is impractical | UI must communicate "processing" states, compensating actions needed |

**Idempotent Event Handlers:** Every event handler must produce the same result if called multiple times with the same event. Use idempotency keys, check-then-act with proper locking, or upsert semantics.

## Vertical Slice Architecture

- Organize code by **feature** rather than by **layer**
- Each slice contains its own handler, validator, model, and data access
- Trade-offs: reduces cross-cutting consistency but improves feature isolation and team autonomy
- Works well with MediatR / CQRS patterns in .NET
- **Detection of need:** When changing a feature requires touching 8+ files across 4+ layers, vertical slices may be more maintainable

## Distributed Systems Fundamentals

### CAP Theorem
- **Consistency + Availability + Partition Tolerance** — pick two (practically: choose CP or AP during network partitions)
- Financial systems typically need CP (consistency over availability during failures)
- Read-heavy content systems can often be AP (eventual consistency acceptable)

### Fallacies of Distributed Computing
1. The network is reliable
2. Latency is zero
3. Bandwidth is infinite
4. The network is secure
5. Topology doesn't change
6. There is one administrator
7. Transport cost is zero
8. The network is homogeneous

**Detection:** Code that calls remote services without timeout, retry, or fallback violates assumptions 1-2. Code that sends large payloads without pagination violates assumption 3.

### Consistency Patterns

| Pattern | Description | Use When |
|---|---|---|
| **Strong consistency** | All readers see latest write immediately | Financial transactions, inventory counts |
| **Eventual consistency** | Readers may see stale data temporarily | Read replicas, caches, search indices |
| **Causal consistency** | Causally related operations seen in order | Chat messages, comment threads |
| **Read-your-writes** | Writer immediately sees their own write | User profile updates, form submissions |

## Data Architecture

### Database per Service
- Each microservice owns its data store exclusively
- No direct database access across services — only via APIs/events
- **Detection of violation:** Multiple services sharing the same database schema, cross-database joins, shared stored procedures

### Transactional Outbox
- Write domain event to an outbox table in the same transaction as the domain change
- A separate process reads the outbox and publishes events
- Guarantees at-least-once delivery without distributed transactions
- **Detection of need:** Service that writes to DB AND publishes message — if either fails independently, data inconsistency results

### Change Data Capture (CDC)
- Capture database changes as events (e.g., Debezium on SQL Server transaction log)
- Useful for syncing data across services without application-level event publishing
- **Detection of need:** Legacy systems that can't be modified to publish events
