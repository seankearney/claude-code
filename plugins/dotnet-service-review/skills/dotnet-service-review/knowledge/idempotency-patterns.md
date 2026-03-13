# Idempotency Patterns

> Shared knowledge module. Referenced by agents that evaluate data integrity, duplicate prevention, and retry safety in distributed services.

---

## Core Principle

An operation is **idempotent** if performing it multiple times produces the same result as performing it once. In distributed systems, network failures, retries, and message redelivery make idempotency a correctness requirement, not an optimization.

**Financial systems require idempotency at every mutation boundary** — API endpoints, message consumers, batch processors, and database writes.

---

## Idempotency Types & Detection Signals

### 1. API Endpoint Idempotency

Mutating HTTP methods (POST, PUT, PATCH) must support client-supplied idempotency keys to prevent duplicate operations from retries or double-clicks.

| What to Look For | Detection Signal |
|---|---|
| `Idempotency-Key` header middleware | No header parsing on POST endpoints |
| `409 Conflict` response for duplicate keys | All POSTs return `200`/`201` regardless of prior execution |
| Server-side key storage with TTL | No idempotency key table or cache |
| Key-scoped locking during processing | Concurrent identical requests both execute |

**In .NET:**
```csharp
// GREEN: Idempotency middleware checks before processing
[HttpPost("payments")]
public async Task<IActionResult> CreatePayment(
    [FromHeader(Name = "Idempotency-Key")] string idempotencyKey,
    [FromBody] PaymentRequest request)
{
    var existing = await _idempotencyStore.GetAsync(idempotencyKey);
    if (existing != null) return StatusCode(existing.StatusCode, existing.Body);
    // ... process and store result keyed by idempotencyKey
}

// RED: No idempotency protection
[HttpPost("payments")]
public async Task<IActionResult> CreatePayment([FromBody] PaymentRequest request)
{
    var result = await _paymentService.Process(request); // duplicate calls = duplicate charges
    return Ok(result);
}
```

### 2. Message/Event Consumer Idempotency

Queue consumers (RabbitMQ, SQS, Kafka) receive messages **at least once**. Redelivery after ack timeout, broker failover, or consumer crash means every handler must tolerate duplicates.

| What to Look For | Detection Signal |
|---|---|
| `MessageId` or `DeduplicationId` tracked in DB | No dedup table, no processed-message ledger |
| Check-before-process pattern | Handler processes every delivery unconditionally |
| Idempotent writes (upsert, not insert) | `INSERT` without `IF NOT EXISTS` or unique constraint |
| Dead letter queue for poison messages | Failed messages silently dropped or infinite retry |

**In .NET (RabbitMQ):**
```csharp
// GREEN: Dedup check before processing
public async Task HandleMessage(PaymentEvent message)
{
    if (await _dedupStore.AlreadyProcessedAsync(message.MessageId))
        return; // safe to ack — already handled

    await _processor.Process(message);
    await _dedupStore.MarkProcessedAsync(message.MessageId);
}

// RED: No dedup — redelivery causes duplicate processing
public async Task HandleMessage(PaymentEvent message)
{
    await _processor.Process(message); // every delivery = another execution
}
```

### 3. Database Operation Idempotency

Write operations must produce the same state whether executed once or multiple times.

| What to Look For | Detection Signal |
|---|---|
| Upsert semantics (`MERGE`, `ON CONFLICT`, `IF NOT EXISTS`) | Blind `INSERT` that fails or duplicates on retry |
| Unique constraints on natural/business keys | Only surrogate key (`IDENTITY`) as uniqueness guard |
| Optimistic concurrency (`rowversion`, ETag, `WHERE version = @v`) | Last-write-wins with no conflict detection |
| Idempotent stored procedures | Procedure assumes single execution, no re-entrancy guard |

**SQL patterns:**
```sql
-- GREEN: Upsert — safe to retry
MERGE INTO Payments AS target
USING (SELECT @PaymentId AS PaymentId) AS source
ON target.PaymentId = source.PaymentId
WHEN NOT MATCHED THEN INSERT (...) VALUES (...);

-- RED: Blind insert — retry creates duplicate row
INSERT INTO Payments (Amount, AccountId, CreatedDate)
VALUES (@Amount, @AccountId, GETDATE());
```

### 4. Retry-Safe Operations

When retry policies (Polly, `Microsoft.Extensions.Http.Resilience`) are configured, the target operation **must** be safe to repeat. Retries on non-idempotent operations cause data corruption.

| What to Look For | Detection Signal |
|---|---|
| Retry policy targets only idempotent operations | Retry wraps payment charge, email send, or file write |
| Side effects guarded by idempotency check | Retry blindly re-executes the full operation |
| Conditional retry (retry on 503, not on 400) | Retry on all exceptions including business errors |

**In .NET (Polly):**
```csharp
// DANGEROUS: Retry on a non-idempotent operation
services.AddHttpClient("PaymentGateway")
    .AddTransientHttpErrorPolicy(p =>
        p.WaitAndRetryAsync(3, attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt))));
// If the gateway charged the card but the response timed out, retry = double charge
```

### 5. Batch/File Processing Idempotency

Reprocessing a batch file (after crash, restart, or rerun) must not create duplicate records.

| What to Look For | Detection Signal |
|---|---|
| File hash or filename tracked in processed-files ledger | No record of which files have been processed |
| Per-record dedup within batch (natural key check) | Each row inserted without existence check |
| Resumable processing (checkpoint after N records) | Crash at row 5000 of 10000 → full reprocessing → 5000 duplicates |
| Atomic commit per batch or per-record | Partial failure leaves orphaned records with no cleanup |

### 6. Saga Compensation Idempotency

In distributed transactions (Saga pattern), compensating actions must themselves be idempotent — a compensation may be retried if the first attempt's result is unknown.

| What to Look For | Detection Signal |
|---|---|
| Compensation checks current state before acting | `RefundPayment` issues refund without checking if already refunded |
| Compensation is a no-op if already compensated | Compensation always executes, causing double-refund |
| Compensation uses the same idempotency key as the original | No correlation between forward and compensating operations |

### 7. Distributed Lock / Check-then-Act

Race conditions occur when multiple processes read-then-write without coordination. This is the **concurrency** dimension of idempotency.

| What to Look For | Detection Signal |
|---|---|
| `SELECT ... FOR UPDATE` or application-level distributed lock | `SELECT` then `INSERT` in separate statements with no lock |
| Optimistic concurrency with version column | Update overwrites without checking if data changed since read |
| Database-level unique constraint as safety net | Only application-level checks (which have TOCTOU gaps) |

### 8. Heuristic vs. True Idempotency

**Critical distinction for financial systems.** Heuristic matching uses business field combinations (amount + date + account) to guess if an operation is a duplicate. True idempotency uses a unique operation identifier.

| Approach | Reliability | Detection Signal |
|---|---|---|
| **True idempotency** (operation ID / idempotency key) | Deterministic — same key = same operation | `IdempotencyKey`, `TransactionId`, `CorrelationId` as unique constraint |
| **Heuristic matching** (amount + date + account) | Probabilistic — legitimate same-amount payments flagged as dupes, or edge-case dupes missed | `SP_GETDUPPAYMENTCOUNT`-style queries matching on business fields |

**Heuristic detection is an anti-pattern in payment systems** — it either blocks legitimate transactions or misses actual duplicates when field combinations coincidentally differ.

---

## Review Checklist (Quick Reference)

When reviewing a service for idempotency, check each layer:

```
[ ] API Layer        — Idempotency-Key on POST/PUT/PATCH endpoints?
[ ] Message Layer    — MessageId dedup on queue consumers?
[ ] Database Layer   — Upsert/MERGE instead of blind INSERT?
[ ] Retry Layer      — Polly/retry targets only idempotent operations?
[ ] Batch Layer      — File/record dedup on reprocessing?
[ ] Saga Layer       — Compensating actions check state before acting?
[ ] Concurrency      — Locks or optimistic concurrency on read-modify-write?
[ ] Dedup Strategy   — True idempotency keys, not heuristic field matching?
```

---

## Severity Classification

| Severity | Condition | Example |
|---|---|---|
| **Critical** | Financial mutation endpoint with no idempotency protection | Payment POST with no idempotency key → duplicate charges |
| **High** | Message consumer without dedup on financial events | RabbitMQ payment handler redelivery → duplicate processing |
| **High** | Retry policy on non-idempotent operation | Polly retry on card charge endpoint |
| **Medium** | Heuristic-only duplicate detection on financial flows | Amount+date matching instead of transaction ID |
| **Medium** | Batch processor with no reprocessing guard | File rerun creates duplicate records |
| **Low** | Non-financial endpoint missing idempotency | Duplicate notification sent |
| **Low** | Missing optimistic concurrency on low-contention data | Profile update race condition |
