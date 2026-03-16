# Testing Gap Analysis & Migration Safety Net Guide

> Reference material for the test analyst agent when performing gap analysis and migration safety net assessment.

## Test Project Detection Patterns

### Naming Conventions

Test projects typically follow these naming patterns:
- `{ProjectName}.Tests`
- `{ProjectName}.UnitTests`
- `{ProjectName}.IntegrationTests`
- `{ProjectName}.FunctionalTests`
- `{ProjectName}.E2ETests`
- `{ProjectName}.Test` (singular)

### Coverage Mapping via ProjectReference

To determine which production project a test project covers:
1. Read the test project's `.csproj` file
2. Look for `<ProjectReference Include="..\..\src\{ProjectName}\{ProjectName}.csproj" />`
3. The referenced project is the production code under test
4. If no `ProjectReference` to a production project exists, the test project may be testing via published packages or is misconfigured

## Cross-Reference Algorithm

For each production class discovered in Step 4b:

1. **Search test directories** for references to the type name using `Grep`
2. **Match criteria** — the type name appears in a test file (`.cs` file within a test project directory) as:
   - A constructor call: `new ClassName(`
   - A variable type: `ClassName sut`
   - A mock setup: `Mock<IClassName>`, `Substitute.For<IClassName>()`
   - A method call: `className.MethodName(`
3. **If no test references the type**, it is a gap
4. **Prioritize gaps by:**
   - **High risk**: Cyclomatic complexity > 10, handles money/PII/auth, or accesses infrastructure (DB, queues, HTTP)
   - **Medium risk**: Moderate complexity (5-10), business logic, or state management
   - **Low risk**: Simple DTOs, constants, configuration POCOs, or thin wrappers

## Migration Safety Net Priorities

When the target framework is .NET Framework 4.x, these test categories are critical before migration, ordered by value:

### 1. API Contract Tests (Highest Priority)
- **Why**: Serialization defaults change between ASP.NET Web API and ASP.NET Core (e.g., camelCase vs PascalCase, `DateTime` formatting, null handling)
- **What to test**: Every API endpoint's request/response shape — HTTP method, route, status codes, JSON structure
- **How to detect gaps**: Count controllers with `[ApiController]` or inheriting `ApiController`/`ControllerBase`, then check for `WebApplicationFactory` or `TestServer` usage in test projects

### 2. Data Access Tests
- **Why**: EF6 and EF Core have different query translation, lazy loading defaults, and `Include` semantics
- **What to test**: Every LINQ query that touches the database, especially `GroupBy`, `Include`, and queries relying on client-side evaluation
- **How to detect gaps**: Find `DbContext`/`ObjectContext` subclasses, count entity types, check for data access tests

### 3. Auth Pipeline Tests
- **Why**: `FormsAuthentication`, `System.Web.Security`, and ASP.NET membership providers have no direct equivalent in ASP.NET Core
- **What to test**: Login flows, role checks, claims transformation, cookie behavior
- **How to detect gaps**: `Grep` for `FormsAuth`, `Membership`, `RoleProvider`, `System.Web.Security`

### 4. Configuration Tests
- **Why**: `ConfigurationManager` and custom config sections don't exist in ASP.NET Core; migration to `IConfiguration` + Options pattern is error-prone
- **What to test**: Every custom config section binding, connection string resolution, environment-specific overrides
- **How to detect gaps**: `Grep` for `ConfigurationManager`, `<section name=`, custom `ConfigurationSection` subclasses

### 5. Infrastructure Adapter Tests
- **Why**: WCF client proxies, MSMQ, COM interop, and other Windows-specific infrastructure need replacement strategies
- **What to test**: Message format compatibility, timeout behavior, error handling for each adapter
- **How to detect gaps**: `Grep` for `ServiceReference`, `MessageQueue`, `DllImport`, `COM` references

## Characterization Testing Patterns

### Snapshot / Golden Master Approach

For capturing current behavior before rewriting:

1. **API snapshots**: Record actual HTTP request/response pairs for every endpoint. Store as `.json` golden files. Assert that current behavior matches snapshots before migration.
2. **Query result snapshots**: Execute EF6 LINQ queries against a seeded test database. Record result sets as golden files. After migration to EF Core, re-run and compare.
3. **Config binding snapshots**: Serialize resolved configuration objects to JSON. Assert structure and values match after migrating to `IConfiguration`.

### When to Use Characterization Tests

- Code has no existing tests and is too complex/risky to understand fully before migration
- Behavior is "correct by definition" (legacy system is the spec)
- Time pressure prevents writing full unit/integration tests before migration begins

## EF6 → EF Core Behavioral Differences

Key differences that tests should guard against:

| Behavior | EF6 | EF Core | Risk |
|----------|-----|---------|------|
| **Lazy loading** | Enabled by default with virtual navigation properties | Disabled by default; requires explicit opt-in via proxies or `ILazyLoader` | Queries that relied on lazy loading silently return null navigations |
| **`Include` semantics** | Ignored for projections (`Select`) | Honored for projections; may change query shape | Different result sets for same LINQ |
| **`GroupBy` translation** | Client-side evaluation as fallback | Throws `InvalidOperationException` if not translatable to SQL | Runtime exceptions on queries that "worked" in EF6 |
| **Client evaluation** | Silently falls back to client-side for untranslatable expressions | Throws by default (configurable) | Runtime exceptions on previously-working queries |
| **`string.Contains`** | Case-sensitive by default | Depends on database collation | Behavioral change in string matching |
| **Owned entities** | Complex types with limited support | Owned entity types with table splitting | Schema and query differences |
| **`DbSet.Add` return** | Returns the entity | Returns `EntityEntry<T>` | Compilation errors if return value was used |
