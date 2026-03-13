# .NET & C# Deep Expertise

> Shared knowledge module. Referenced by agents that evaluate .NET codebases.

## Modern C# Features (8-13)

### C# 8 (with .NET Core 3.0+ / .NET Standard 2.1)
- **Nullable reference types:** `string?` vs `string`. Enable with `<Nullable>enable</Nullable>`. Detect missing null checks when disabled.
- **Pattern matching:** `switch` expressions, property patterns, tuple patterns
- **Default interface methods:** Interface methods with implementation. Use sparingly.
- **`using` declarations:** `using var stream = ...` — disposed at end of scope
- **Async streams:** `IAsyncEnumerable<T>` with `await foreach`

### C# 9 (with .NET 5)
- **Records:** `record Person(string Name, int Age)` — immutable reference types with value equality
- **Init-only properties:** `public string Name { get; init; }` — set only during initialization
- **Top-level statements:** Single-file programs without boilerplate
- **Target-typed `new`:** `List<string> items = new()` — reduces redundancy

### C# 10 (with .NET 6)
- **Global using directives:** `global using System.Linq;` — reduces per-file noise
- **File-scoped namespaces:** `namespace Foo;` — reduces indentation
- **Record structs:** Value-type records for performance-sensitive scenarios
- **Constant interpolated strings:** `const string s = $"{nameof(Foo)}.bar"`

### C# 11 (with .NET 7)
- **Raw string literals:** `"""multiline"""` — no escaping needed
- **Required members:** `required` modifier ensures initialization
- **Generic math:** `INumber<T>` — generic numeric operations
- **List patterns:** `[1, 2, .., 9]` — match collection shapes

### C# 12 (with .NET 8)
- **Primary constructors:** `class Foo(ILogger logger)` — DI-friendly
- **Collection expressions:** `int[] x = [1, 2, 3]`
- **Default lambda parameters:** `(int x = 5) => x * 2`
- **Alias any type:** `using Point = (int X, int Y)`

### C# 13 (with .NET 9)
- **`params` collections:** `params ReadOnlySpan<int>` — stack allocation
- **`Lock` type:** New `System.Threading.Lock` with scoped locking
- **Overload resolution priority:** `[OverloadResolutionPriority]` attribute

## ASP.NET Core Internals

### Middleware Pipeline
- Request flows through middleware in order; response flows back in reverse
- Order matters: `UseAuthentication()` before `UseAuthorization()` before `MapControllers()`
- **Detection:** Middleware in wrong order, missing error handling middleware, no request logging

### Dependency Injection Lifetimes

| Lifetime | Scope | Pitfall |
|---|---|---|
| **Transient** | New instance every request | Expensive objects created repeatedly |
| **Scoped** | One per HTTP request | Shared across the request pipeline |
| **Singleton** | One for application lifetime | Must be thread-safe, no scoped dependencies |

**Captive Dependency Anti-Pattern:** Singleton service depends on Scoped service → Scoped service lives forever as Singleton, causing data staleness and potential thread-safety issues.

```csharp
// BAD: Singleton captures Scoped dependency
services.AddSingleton<IMyService, MyService>();    // Singleton
services.AddScoped<IDbContext, AppDbContext>();     // Scoped — but captured by Singleton!
```

**Detection:** Singleton registrations that inject Scoped services. Check `Program.cs`/`Startup.cs` registrations.

### IOptions Pattern

| Interface | Behavior | Use When |
|---|---|---|
| `IOptions<T>` | Singleton, read once at startup | Static config that never changes |
| `IOptionsSnapshot<T>` | Scoped, reloads per request | Config that may change between requests |
| `IOptionsMonitor<T>` | Singleton, notifies on change | Long-lived services that need live config updates |

**Detection:** `Configuration.GetValue<string>("key")` scattered through code — should use strongly-typed options.

### Health Checks
- `AddHealthChecks()` with custom checks for each dependency
- Map to endpoints: `/health/live`, `/health/ready`
- **Detection:** No health check registration, health checks that always pass, missing dependency checks

### Rate Limiting (.NET 7+)
- Built-in: Fixed window, sliding window, token bucket, concurrency limiter
- **Detection:** No rate limiting on public APIs, especially authentication endpoints

## .NET Framework 4.8 Legacy Patterns

### HttpClient Misuse

```csharp
// BAD: Creates new HttpClient per request — socket exhaustion
using (var client = new HttpClient()) { ... }

// BAD: Static HttpClient — DNS changes not respected
private static readonly HttpClient _client = new HttpClient();

// GOOD (.NET Core): IHttpClientFactory
services.AddHttpClient<IMyService, MyService>();

// ACCEPTABLE (Framework 4.8): Static with SocketsHttpHandler or ServicePointManager
```

**Detection:** `new HttpClient()` in using blocks, especially in loops or per-request code.

### ConfigurationManager
- Framework 4.8: `ConfigurationManager.AppSettings["key"]` — string-based, no type safety
- `ConfigurationManager.ConnectionStrings["name"]` — check for hardcoded values
- **Detection:** Magic strings for config keys, no validation of required config at startup

### Async Anti-Patterns

| Anti-Pattern | Code | Problem |
|---|---|---|
| **Sync-over-async** | `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` | Deadlock in ASP.NET (SyncContext), thread pool starvation |
| **Async void** | `async void OnClick(...)` | Exceptions crash the process, can't be awaited |
| **Missing ConfigureAwait** | `await Task.Delay(1000)` (no `.ConfigureAwait(false)`) | Deadlock in Framework 4.8 with SyncContext |
| **Fire-and-forget** | `_ = DoSomethingAsync()` | Unobserved exceptions, no error handling |

### ThreadPool Starvation
- CLR injects new threads at ~1 per 500ms when pool is exhausted
- Sync-over-async blocks threads, preventing async completions
- **Detection:** `ThreadPool.SetMinThreads()` calls (band-aid for starvation), `.Result`/`.Wait()` calls

## Entity Framework / Dapper

### EF Core Anti-Patterns

| Anti-Pattern | Detection | Fix |
|---|---|---|
| **N+1 queries** | Loop with lazy-loaded navigation properties | Use `.Include()` or projection with `.Select()` |
| **Missing AsNoTracking** | Read-only queries without `.AsNoTracking()` | Add for queries that don't modify entities |
| **Overfetching** | `ToList()` then filter in memory | Apply `.Where()` before materializing |
| **Long-lived DbContext** | Singleton-registered `DbContext` | Register as Scoped |
| **No query filters** | Soft-delete without global filter | Use `HasQueryFilter(e => !e.IsDeleted)` |

### Dapper Patterns
- Parameterized queries: `connection.Query("SELECT * FROM Users WHERE Id = @Id", new { Id = id })`
- **Detection:** String interpolation in SQL, missing `using` on connections, no command timeout

### Connection Pool Exhaustion
- **Detection:** `new SqlConnection()` without `using`, `CommandTimeout = int.MaxValue`, connections opened but never closed
- Default pool size: 100 connections. Exhaustion causes `SqlException: Timeout expired`

## Performance Patterns

### Span<T> and Memory<T>
- `Span<T>`: Stack-only, zero-allocation slicing of arrays/strings
- `Memory<T>`: Heap-safe version of Span for async contexts
- **Use when:** Parsing, string manipulation, buffer processing in hot paths
- **Detection of need:** `string.Substring()`, `Array.Copy()`, `ToArray()` in performance-critical loops

### Object Pooling
- `ArrayPool<T>.Shared.Rent(size)` — reuse arrays without GC pressure
- `ObjectPool<T>` — reuse expensive objects (StringBuilder, etc.)
- **Detection:** `new byte[largeSize]` in loops, frequent large allocations in hot paths

### ValueTask
- `ValueTask<T>`: Avoids Task allocation when result is often synchronous (cache hits)
- **Rule:** NEVER await a ValueTask twice. NEVER use `.Result` on incomplete ValueTask.
- **Use when:** Method often returns cached/synchronous results. Otherwise, use `Task<T>`.

### Channels
- `Channel<T>`: High-performance producer-consumer pattern
- Bounded channels provide backpressure
- **Detection of need:** `ConcurrentQueue` + polling loop, `BlockingCollection` in async code

## Diagnostics

### Activity and DiagnosticSource
- `Activity`: Built-in distributed tracing primitive in .NET
- `ActivitySource`: Creates activities (OpenTelemetry-compatible)
- `DiagnosticSource`: Rich diagnostic events with payload (compile-time typed)
- **Detection:** Custom correlation ID headers instead of W3C `traceparent`, no `ActivitySource` usage

### Metrics
- `System.Diagnostics.Metrics`: Built-in metrics API (.NET 6+)
- `Counter<T>`, `Histogram<T>`, `UpDownCounter<T>`, `ObservableGauge<T>`
- **Detection:** No application metrics, metrics via string-based EventCounters only

### Structured Logging
```csharp
// GOOD: Structured — searchable by OrderId
_logger.LogInformation("Processing order {OrderId} with {ItemCount} items", orderId, items.Count);

// BAD: String interpolation — not searchable
_logger.LogInformation($"Processing order {orderId} with {items.Count} items");

// BAD: Concatenation — not searchable, potential perf issue
_logger.LogInformation("Processing order " + orderId);
```

**Detection:** `$"..."` or `+` in log method calls, `Console.WriteLine` instead of `ILogger`, missing log levels.
