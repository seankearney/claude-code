# Service Review Rubric

> Scoring criteria for the dotnet-service-review agent's formal repository health reviews.

---

## Evaluation Categories

Score each category as 🟢 Green, 🟠 Orange, or 🔴 Red based on these criteria:

### Category 1: Code Quality & Design

*SOLID principles, clean code, proper patterns*

| Criterion             | 🟢 Green                                          | 🟠 Orange                            | 🔴 Red                                       |
| --------------------- | ------------------------------------------------ | ----------------------------------- | ------------------------------------------- |
| Single Responsibility | Classes/methods have one reason to change        | Some god classes or bloated methods | Widespread violation, monolithic structures |
| Dependency Injection  | DI container used, abstractions over concretions | Partial DI, some hard dependencies  | No DI, tightly coupled components           |
| Naming Clarity        | Intent-revealing names, consistent conventions   | Inconsistent naming, some ambiguity | Cryptic names, abbreviations, misleading    |
| Method Size           | Methods < 20 lines, single abstraction level     | Some long methods (20-50 lines)     | Methods > 50 lines, mixed abstraction       |
| Code Duplication      | DRY principle followed                           | Minor duplication                   | Significant copy-paste code                 |

### Category 2: .NET Practices

*Architecture, DI, configuration, static analysis, performance, security, testing*

| Criterion                              | 🟢 Green                                                                                                            | 🟠 Orange                                                                     | 🔴 Red                                                                                            |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------- |
| Static Code Analysis                   | Zero suppressed analyzer warnings, nullable enabled, proper async/await, specific exception handling                | Some analyzer suppressions, partial nullable adoption, occasional sync-over-async | Analyzers disabled or widely suppressed, `catch (Exception) { }` swallowing errors, blocking async calls |
| Architecture & Project Structure       | Clear layer separation (API/Domain/Infrastructure), dependency inversion enforced, no circular references           | Some layer bleeding, inconsistent boundaries                                   | Monolithic projects, no discernible structure, circular dependencies                              |
| Dependency Injection & Configuration   | Correct service lifetimes, IOptions pattern, strongly-typed config, no `new` for services                           | Some magic strings, occasional lifetime mismatches                             | Hardcoded values, manual instantiation, no environment separation, service locator anti-pattern   |
| Performance & Resource Management      | Proper IDisposable/IAsyncDisposable, connection pooling, no thread pool starvation from sync-over-async             | Missing some `Dispose` calls, no caching strategy                              | Resource leaks, unbounded allocations, no pooling or streaming for large data                     |
| Idempotency & Data Integrity           | Idempotency keys on mutating API endpoints, dedup on message consumers, upsert semantics on writes, retry-safe operations | Partial coverage — some endpoints protected, no message dedup, heuristic-only duplicate detection | No idempotency protection on payment/financial endpoints, retries on non-idempotent operations, blind inserts without dedup |

### Category 3: Testing

*Test presence, coverage, quality*

| Criterion         | 🟢 Green                                            | 🟠 Orange                        | 🔴 Red                         |
| ----------------- | -------------------------------------------------- | ------------------------------- | ----------------------------- |
| Test Presence     | Unit tests exist for business logic                | Some tests, incomplete coverage | No tests                      |
| Test Organization | Arrange-Act-Assert, descriptive names              | Inconsistent structure          | Tests hard to understand      |
| Test Independence | Tests isolated, no shared state                    | Some ordering dependencies      | Tests must run in sequence    |
| Mocking Strategy  | Interfaces mocked, no infrastructure in unit tests | Some infrastructure leakage     | Tests hit real databases/APIs |
| Coverage          | >70% meaningful coverage                           | 40-70% coverage                 | <40% or unmeasured            |

### Category 4: Security

*Secrets, vulnerabilities, auth patterns*

| Criterion                  | 🟢 Green                                       | 🟠 Orange                        | 🔴 Red                                      |
| -------------------------- | --------------------------------------------- | ------------------------------- | ------------------------------------------ |
| No Secrets in Code         | No hardcoded credentials, uses vault/env vars | Some legacy secrets (rotated)   | Active secrets in repository |
| Dependency Vulnerabilities | No known CVEs, regular updates                | Low-severity CVEs               | **CRITICAL: High/Critical CVEs unpatched** |
| Authentication             | Industry-standard auth (OAuth, OIDC, JWT)     | Basic auth with proper handling | Custom auth, weak patterns                 |
| Authorization              | Role-based or policy-based access             | Some authorization gaps         | No authorization checks                    |
| Input Validation           | All inputs validated, parameterized queries   | Partial validation              | SQL injection, XSS risks                   |

### Category 5: Maintainability

*Dependencies, activity, structure*

| Criterion            | 🟢 Green                      | 🟠 Orange                  | 🔴 Red                                |
| -------------------- | ---------------------------- | ------------------------- | ------------------------------------ |
| Dependency Freshness | Dependencies < 1 year old    | Some outdated (1-2 years) | Major versions behind (>2 years)     |
| Last Activity        | Commits within 6 months      | 6-12 months stale         | >12 months no activity               |
| Solution Structure   | Logical project organization | Some inconsistency        | Chaotic or no structure              |
| Build Success        | Builds without errors        | Warnings present          | Build broken or unclear how to build |

---

## Informational Categories (Not Scored)

### Documentation

*README, setup instructions, code clarity — assessed for awareness but does not affect overall health score*

| Criterion          | 🟢 Green                                       | 🟠 Orange                     | 🔴 Red                            |
| ------------------ | --------------------------------------------- | ---------------------------- | -------------------------------- |
| README             | Purpose, setup, usage clearly documented      | Basic README exists          | No README or severely outdated   |
| Setup Instructions | Can get running from README alone             | Some tribal knowledge needed | No idea how to run it            |
| Code Comments      | Self-documenting code, comments explain *why* | Over-commented or sparse     | Misleading or commented-out code |
| API Documentation  | Swagger/OpenAPI for APIs                      | Partial documentation        | No API docs for public endpoints |

---

## Critical Failures (Auto-Red Overall)

Any of these trigger an automatic 🔴 Red overall health score:

- Critical security vulnerabilities (unpatched CVEs)
- No tests AND no discernible architecture
- Abandoned with no documentation

---

## Health Score Calculation

**Based on 5 scored categories** (Code Quality, .NET Practices, Testing, Security, Maintainability). Documentation is informational only and does not affect the overall score.

```
🟢 GREEN  = 0 Red categories AND ≤2 Orange categories
🟠 ORANGE = 1-2 Red categories OR 3+ Orange categories
🔴 RED    = 3+ Red categories OR any Critical failure
```
