# SOLID Principles, Design Patterns & Refactoring

> Shared knowledge module. Referenced by agents that evaluate code quality, recommend refactoring, and assess architectural fitness.

---

## SOLID Principles

- **Single Responsibility (SRP):** Each class/module has one reason to change. Detect god classes, mixed concerns, and feature envy. A class handling HTTP requests AND database queries AND email sending violates SRP.
- **Open/Closed (OCP):** Extensions via abstraction, not modification. Look for strategy pattern opportunities vs. switch/if chains. Adding a new payment type should not require modifying existing payment processing code.
- **Liskov Substitution (LSP):** Subtypes must be substitutable. Detect violated contracts, incorrect inheritance hierarchies, and exceptions thrown from overrides that break caller expectations. A `ReadOnlyRepository` that throws `NotSupportedException` on `Save()` violates LSP.
- **Interface Segregation (ISP):** Prefer role-specific interfaces over fat interfaces. Flag `IService` interfaces with 15+ methods. Prefer `IReader`, `IWriter`, `IDeleter` over `ICrudRepository`.
- **Dependency Inversion (DIP):** High-level modules depend on abstractions. Detect `new` of concrete dependencies inside business logic, static helper abuse, and missing DI registration. Business logic should never `new up` an `HttpClient` or `SqlConnection` directly.

---

## Design Patterns (GoF & Beyond)

For each pattern: what it does, **code smells that indicate you need it**, when to use, when NOT to use.

### Creational Patterns

#### Factory Method / Abstract Factory

**Purpose:** Decouple object creation from usage. Caller requests an object via interface; factory decides which concrete type to return.

**Code smells indicating need:**
- `new ConcreteType()` inside business logic, especially in `if/switch` chains selecting which type to create
- Adding a new product type requires modifying existing creation code (violates OCP)
- Constructor parameters differ across product types

**When to use:** Object creation logic is complex or varies by context. Multiple related objects must be created together (Abstract Factory).

**When NOT to use:** Only one implementation exists and is unlikely to change. Simple `new` is fine — don't create a factory for a factory's sake.

#### Builder

**Purpose:** Construct complex objects step by step, separating construction from representation.

**Code smells indicating need:**
- Constructors with 5+ parameters ("telescoping constructor")
- Multiple constructor overloads with optional parameters
- Object requires multi-step initialization that callers often get wrong

**When to use:** Object has many optional configuration parameters. Construction requires validation across multiple fields.

**When NOT to use:** Object has 1-3 required fields and no optional configuration. A simple constructor or record type suffices.

#### Prototype

**Purpose:** Create new objects by cloning an existing instance rather than constructing from scratch.

**Code smells indicating need:**
- Expensive object initialization (DB lookup, complex calculation) repeated for similar objects
- `new` + copy every field manually

**When to use:** Object creation is expensive and most instances share similar state. Deep copy with selective modifications.

**When NOT to use:** Objects are cheap to create. Shallow vs. deep copy semantics are ambiguous.

#### Singleton

**Purpose:** Ensure a class has one instance and provide global access to it.

**Code smells indicating need:** Truly global, shared resources (connection pool manager, logging infrastructure).

**When to use:** Almost never in modern .NET — use DI container lifetime management (`AddSingleton<T>()`) instead.

**When NOT to use:** For anything that should be testable or configurable. Manual Singletons hide dependencies and prevent testing. Registering as Singleton with Scoped dependencies = **captive dependency** anti-pattern.

#### Object Pool

**Purpose:** Reuse expensive objects instead of creating/destroying them repeatedly.

**Code smells indicating need:**
- `new byte[largeSize]` in loops (GC pressure)
- `new HttpClient()` per request (socket exhaustion)
- Frequent creation/disposal of expensive resources

**When to use:** Object creation is measurably expensive. Pool members are stateless or easily resettable.

**When NOT to use:** Objects are cheap. Pool management complexity exceeds creation cost. Use framework facilities (`IHttpClientFactory`, `ArrayPool<T>`) over hand-rolled pools.

---

### Structural Patterns

#### Adapter

**Purpose:** Convert one interface to another that clients expect. Bridge between incompatible interfaces.

**Code smells indicating need:**
- Third-party library API doesn't match your domain interfaces
- Legacy system returns data in a format your code can't consume directly
- Multiple implementations of the same concept with different APIs

**When to use:** Integration boundaries — wrapping external APIs, legacy systems, or third-party libraries behind your own interface.

**When NOT to use:** Interfaces are already compatible. Don't create adapters between your own classes that you control.

#### Bridge

**Purpose:** Separate an abstraction from its implementation so both can vary independently.

**Code smells indicating need:**
- Class hierarchy explosion — `WindowsButton`, `MacButton`, `LinuxButton` × `RoundButton`, `SquareButton`
- Platform-specific code mixed with business logic

**When to use:** Abstraction and implementation should evolve independently. Multiple dimensions of variation.

**When NOT to use:** Only one dimension of variation exists — use simple inheritance or Strategy instead.

#### Composite

**Purpose:** Treat individual objects and compositions uniformly via a common interface.

**Code smells indicating need:**
- Tree structures where operations must apply to both leaves and branches
- Recursive `if (isLeaf) ... else foreach(children)` scattered through code
- File system, org chart, menu, or expression tree structures

**When to use:** Part-whole hierarchies where clients should treat individual and composite objects uniformly.

**When NOT to use:** Structure is flat (no nesting). Leaf and composite behaviors are fundamentally different.

#### Decorator

**Purpose:** Add behavior to objects dynamically without modifying them. Chain of wrappers.

**Code smells indicating need:**
- Cross-cutting concerns (logging, caching, retry, timing) copy-pasted into every method
- Subclass explosion to combine optional behaviors
- Feature flags that add/remove behaviors at runtime

**When to use:** Adding optional behaviors to existing objects. Combining behaviors in different permutations. ASP.NET middleware IS a decorator chain.

**When NOT to use:** Behavior is always needed (just put it in the class). Deep nesting of decorators makes debugging difficult.

#### Facade

**Purpose:** Provide a simplified interface to a complex subsystem.

**Code smells indicating need:**
- Controller/service orchestrates 5+ dependencies to complete one operation
- Client code must call multiple subsystem classes in specific order
- Tight coupling between client and subsystem internals

**When to use:** Complex subsystem with many moving parts. Clients need a simpler entry point.

**When NOT to use:** Subsystem is already simple. Facade becomes a god object that knows too much.

#### Flyweight

**Purpose:** Share common state between many objects to reduce memory footprint.

**Code smells indicating need:**
- Millions of similar objects consuming excessive memory
- Objects have large intrinsic (shared) state and small extrinsic (unique) state

**When to use:** Large numbers of similar objects where shared state can be extracted. Game engines, text rendering, geographic data.

**When NOT to use:** Object count is small. State is mostly unique per instance. Complexity of flyweight management exceeds memory savings.

#### Proxy

**Purpose:** Control access to an object through a surrogate. Adds a layer of indirection.

**Code smells indicating need:**
- Need to add access control, lazy loading, logging, or caching without changing the real object
- Remote objects that need local representation
- Expensive object initialization that should be deferred

**Types:** Virtual proxy (lazy loading), protection proxy (access control), remote proxy (network access), caching proxy.

**When NOT to use:** Direct access is fine. Proxy adds latency and complexity for no benefit.

---

### Behavioral Patterns

#### Strategy

**Purpose:** Define a family of interchangeable algorithms. Client selects at runtime.

**Code smells indicating need:**
- `switch` or `if/else` chains selecting behavior based on type/enum
- Multiple classes that differ only in one algorithm
- Need to change algorithm at runtime
- Adding a new behavior requires modifying existing code (OCP violation)

**When to use:** Multiple algorithms for the same task. Algorithm selection varies by context, customer, or configuration.

**When NOT to use:** Only one algorithm exists. Algorithms are so different they don't share an interface.

**Detection:** `switch (paymentType)` blocks that select processing logic → candidate for Strategy per payment type.

#### Observer

**Purpose:** Define one-to-many dependency so dependents are notified of state changes.

**Code smells indicating need:**
- Method calls a list of callbacks/handlers after state change
- Tight coupling between publisher and subscribers
- Need to add new notification receivers without modifying the publisher

**In .NET:** `event`/delegates, `IObservable<T>`, domain events via MediatR `INotification`.

**When NOT to use:** Single subscriber with a known, stable relationship. Event storms from cascading notifications.

#### Command

**Purpose:** Encapsulate a request as an object. Enables undo, queuing, logging, and deferred execution.

**Code smells indicating need:**
- Operations that need to be undoable, queueable, or logged
- Different UI elements trigger the same operation
- Need to decouple request issuer from executor

**When to use:** CQRS, undo/redo, job queues, macro recording. MediatR `IRequest` IS the Command pattern.

**When NOT to use:** Simple, synchronous operations that don't need queuing or undo.

#### Chain of Responsibility

**Purpose:** Pass a request along a chain of handlers until one handles it.

**Code smells indicating need:**
- Nested `if/else` blocks checking conditions in sequence
- Need to dynamically add/remove/reorder processing steps
- Multiple handlers may process the same request

**In .NET:** ASP.NET middleware pipeline, `DelegatingHandler` for HTTP, validation chains.

**When NOT to use:** Only one handler ever processes the request. Order doesn't matter (use Observer instead).

#### Mediator

**Purpose:** Define an object that encapsulates how objects interact. Reduces direct dependencies.

**Code smells indicating need:**
- Classes with 10+ dependencies (many of which are just "pass-through" to other services)
- Circular dependencies between classes
- Adding a new feature requires modifying many existing classes

**In .NET:** MediatR library. Warning: overuse creates hidden coupling ("MediatR everywhere" anti-pattern).

**When to use:** Complex interactions between many classes. Request/response pipelines with cross-cutting concerns.

**When NOT to use:** Simple direct calls between 2-3 classes. Don't use MediatR to replace `service.Method()` with `mediator.Send(new Command())` — that adds complexity without value.

#### State

**Purpose:** Object changes behavior when its internal state changes. Appears to change its class.

**Code smells indicating need:**
- Large `switch` statements on state/status fields that determine behavior
- State-dependent behavior duplicated across multiple methods
- Adding a new state requires modifying multiple places

**When to use:** Object has distinct states with different behavior in each. Payment processing (Pending → Authorized → Captured → Settled), order lifecycle, workflow engines.

**When NOT to use:** Simple boolean flags with minimal behavioral differences. Two states don't justify the pattern.

**Detection:** Payment status columns used as state machines in database tables → candidate for explicit State pattern.

#### Template Method

**Purpose:** Define algorithm skeleton in a base class; let subclasses override specific steps.

**Code smells indicating need:**
- Duplicated algorithm structures across classes with minor step variations
- Copy-paste code where only 1-2 methods differ between copies

**When to use:** Several classes follow the same algorithm with different implementations of specific steps.

**When NOT to use:** Algorithm variations are extensive (most steps differ) — use Strategy instead. Avoid deep inheritance hierarchies.

#### Iterator

**Purpose:** Provide sequential access to collection elements without exposing underlying structure.

**In .NET:** `IEnumerable<T>` / `IAsyncEnumerable<T>` — the pattern is built into the language via `foreach` and LINQ.

**When to use:** Custom collections, streaming data, lazy evaluation.

**When NOT to use:** Standard `IEnumerable<T>` already exists for your scenario. Don't hand-roll iterators.

#### Visitor

**Purpose:** Add new operations to existing class hierarchies without modifying them.

**Code smells indicating need:**
- Frequent need to add new operations across a fixed set of types
- `switch` on type to perform different operations per type (and you can't add a method to the types)

**When to use:** Fixed type hierarchy (AST nodes, document elements) with frequently changing operations (rendering, serialization, validation).

**When NOT to use:** Type hierarchy changes frequently. Double-dispatch complexity is not justified for simple operations.

#### Memento

**Purpose:** Capture and externalize object state for later restoration (undo).

**When to use:** Undo/redo, transaction rollback, snapshotting.

**When NOT to use:** State is trivially reconstructible. Saving full state snapshots is too expensive.

---

### Enterprise Patterns

- **Repository:** Abstract data access behind a collection-like interface. Detect repositories that leak `IQueryable` (breaks abstraction) or generic repositories that add no value over the ORM. See `database-patterns.md` for depth.
- **Unit of Work:** Coordinate multiple repository operations in a single transaction. Detect manual transaction management scattered across services. See `database-patterns.md`.
- **Specification:** Encapsulate query criteria as composable objects. Detect repeated query logic (same WHERE clauses) scattered across services.
- **Domain Events:** Decouple side effects from core operations. Detect methods that mix core logic with notifications/logging/auditing inline.
- **Saga / Process Manager:** Coordinate long-running distributed transactions with compensation. Detect distributed operations without rollback/compensation logic.

---

## Code Smells → Pattern Mapping

Quick reference: when you detect a code smell, which pattern(s) address it?

| Code Smell | Indicates Need For |
|---|---|
| God class (1000+ lines, 10+ deps) | SRP extraction, Facade, Mediator |
| Switch on type/enum for behavior | Strategy, State, or polymorphism |
| Switch on type for creation | Factory Method, Abstract Factory |
| Copy-paste algorithm with minor variations | Template Method or Strategy |
| Cross-cutting concerns (logging/cache/retry) in every method | Decorator, Chain of Responsibility |
| 5+ constructor parameters | Builder, or SRP extraction |
| Nested if/else checking conditions in sequence | Chain of Responsibility |
| Class changes behavior based on status field | State |
| `new ConcreteType()` scattered in business logic | Factory, DI (Dependency Inversion) |
| Multiple classes react to one event | Observer, Domain Events |
| Complex subsystem with many entry points | Facade |
| Tight coupling between many classes | Mediator |
| Database used as message queue (status polling) | Replace with proper messaging pattern (see `database-patterns.md`) |
| Stored procedures containing business logic | Domain model, move logic to application layer |

---

## Refactoring Patterns

When recommending fixes, use these named refactoring techniques:

### Composing Methods
| Refactoring | When | Technique |
|---|---|---|
| **Extract Method** | Long method, comment explaining a block | Pull block into named method |
| **Inline Method** | Method body is as clear as the name | Replace call with body |
| **Replace Temp with Query** | Local variable holds result of an expression used once | Extract expression to method |

### Moving Features
| Refactoring | When | Technique |
|---|---|---|
| **Move Method** | Method uses more features of another class | Move to the class it belongs to |
| **Extract Class** | Class does two things | Split into two classes |
| **Inline Class** | Class does almost nothing | Merge into another class |

### Organizing Data
| Refactoring | When | Technique |
|---|---|---|
| **Replace Magic Number with Constant** | Literal values scattered in code | Extract to named constant |
| **Encapsulate Field** | Public field accessed directly | Make private, add property |
| **Replace Type Code with Strategy/State** | Behavior varies by type code field | Extract to Strategy or State pattern |

### Simplifying Conditionals
| Refactoring | When | Technique |
|---|---|---|
| **Decompose Conditional** | Complex `if` condition | Extract condition and branches to methods |
| **Replace Nested Conditionals with Guard Clauses** | Deep nesting | Return early for special cases |
| **Replace Conditional with Polymorphism** | `switch` on type selects behavior | Use inheritance/interfaces + Strategy |

### Simplifying Method Calls
| Refactoring | When | Technique |
|---|---|---|
| **Introduce Parameter Object** | Multiple params always passed together | Group into a class/record |
| **Replace Parameter with Method Call** | Caller computes value that callee could compute itself | Let callee get its own data |
| **Preserve Whole Object** | Passing several fields from same object | Pass the object instead |

---

## Anti-Patterns to Flag

| Anti-Pattern | Detection Signal | Impact |
|---|---|---|
| **Service Locator** | `IServiceProvider.GetService<T>()` in business logic | Hides dependencies, untestable |
| **Ambient Context** | `static` current context (`HttpContext.Current`, `Thread.CurrentPrincipal`) | Thread-unsafe, untestable, hidden coupling |
| **Anemic Domain Model** | Domain objects are pure data bags; all logic in services | Procedural code wearing OO clothes |
| **God Object** | Class with 1000+ lines, 20+ methods, 10+ dependencies | Unmaintainable, untestable |
| **Lava Flow** | Dead code, commented-out blocks, unused classes left "just in case" | Noise, maintenance burden |
| **Golden Hammer** | Same pattern/tool for every problem (e.g., stored procedures for everything) | Misfit solutions, missed opportunities |
| **Cargo Cult** | Patterns applied without understanding (Repository over EF that just wraps DbSet) | Complexity without value |
| **Primitive Obsession** | Using strings/ints for domain concepts (customerId as `string`, money as `decimal`) | No type safety, validation scattered |
| **Feature Envy** | Method accesses another class's data more than its own | Belongs in the other class |
| **Data Clumps** | Same group of fields/parameters always appear together | Should be a class/record |
| **Shotgun Surgery** | One change requires editing many classes | Missing abstraction, tight coupling |
| **Divergent Change** | One class changed for many different reasons | Violates SRP, should be split |
