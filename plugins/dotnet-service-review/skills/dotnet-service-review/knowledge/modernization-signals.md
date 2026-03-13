# Modernization Signals

> Reference knowledge for the dotnet-modernization-analyst agent. Defines rubrics for Migration Complexity and Runtime & Deployment assessment.

---

## Migration Complexity Rubric

Score using a "worst significant signal" approach — the overall Migration Complexity score is determined by the worst signal that represents a significant portion of the codebase (not a single isolated occurrence).

### Target Framework

| 🟢 Green | 🟠 Orange | 🔴 Red |
|----------|----------|--------|
| .NET 6+ (modern SDK-style) | .NET Core 3.1 or .NET 5 (EOL but modern SDK) | .NET Framework 4.x or earlier |

### Package Format

| 🟢 Green | 🟠 Orange | 🔴 Red |
|----------|----------|--------|
| PackageReference in all projects | Mix of PackageReference and packages.config | packages.config only |

### Framework-Only API Usage

| 🟢 Green | 🟠 Orange | 🔴 Red |
|----------|----------|--------|
| No framework-only APIs detected | Minor usage (≤3 framework-only APIs, all have direct replacements) | Heavy usage (>3 framework-only APIs, or any without clear replacement) |

### NuGet Compatibility

| 🟢 Green | 🟠 Orange | 🔴 Red |
|----------|----------|--------|
| All packages available for .NET 6+ | Most packages available, 1-2 need replacement | Multiple packages with no .NET 6+ equivalent |

### API Surface Area

| 🟢 Green | 🟠 Orange | 🔴 Red |
|----------|----------|--------|
| ≤10 controllers/endpoints, standard patterns | 11-30 controllers/endpoints, or mixed patterns (MVC + Web API) | >30 controllers/endpoints, or heavy use of non-portable patterns (ASMX, WCF services) |

### Data Access

| 🟢 Green | 🟠 Orange | 🔴 Red |
|----------|----------|--------|
| EF Core or Dapper (cross-platform ORMs) | EF6 (migration path exists) or mixed EF6 + ADO.NET | Heavy ADO.NET with stored proc coupling, DataSet/DataTable patterns, or typed datasets |

### Configuration

| 🟢 Green | 🟠 Orange | 🔴 Red |
|----------|----------|--------|
| appsettings.json / IConfiguration | Mix of appsettings.json and web.config / app.config | web.config / app.config only, heavy use of ConfigurationManager, custom config sections |

---

## Runtime & Deployment Rubric

### Hosting Model

| 🟢 Green | 🟠 Orange | 🔴 Red |
|----------|----------|--------|
| Kestrel / self-hosted / containerized | IIS with modern pipeline (ASP.NET Core on IIS) | Classic IIS pipeline (System.Web), Windows Service with ServiceBase, OWIN self-host |

### Containerization Readiness

| 🟢 Green | 🟠 Orange | 🔴 Red |
|----------|----------|--------|
| Dockerfile present, or stateless design | No Dockerfile but no obvious blockers | Stateful local dependencies (local file system state, machine-specific config, GAC dependencies) |

### CI/CD Pipeline

| 🟢 Green | 🟠 Orange | 🔴 Red |
|----------|----------|--------|
| Modern CI/CD (GitHub Actions, Azure DevOps YAML, GitLab CI) | Legacy CI/CD (TeamCity, Jenkins, Azure DevOps classic) | No CI/CD pipeline detected |

### Windows-Only Dependencies

| 🟢 Green | 🟠 Orange | 🔴 Red |
|----------|----------|--------|
| No Windows-only dependencies | Minor Windows deps with workarounds (Registry reads → config, EventLog → Serilog) | Blocking Windows deps (P/Invoke to native DLLs, COM interop, WPF/WinForms, MSMQ, Windows Auth with no alternative) |

### Platform Target

| 🟢 Green | 🟠 Orange | 🔴 Red |
|----------|----------|--------|
| AnyCPU or platform-agnostic | x86/x64 specified but no native deps | Native dependencies (C++/CLI, P/Invoke to custom DLLs) |

---

## Modernization Readiness Composite Matrix

The two sub-scores combine into a composite Modernization Readiness score:

| Migration Complexity | Runtime & Deployment | Modernization Readiness |
|---------------------|---------------------|------------------------|
| 🟢 | 🟢 | 🟢 |
| 🟢 | 🟠 | 🟢 |
| 🟠 | 🟢 | 🟠 |
| 🟢 | 🔴 | 🟠 |
| 🟠 | 🟠 | 🟠 |
| 🔴 | 🟢 | 🟠 |
| 🟠 | 🔴 | 🔴 |
| 🔴 | 🟠 | 🔴 |
| 🔴 | 🔴 | 🔴 |

**Interpretation:**
- 🟢 **Ready** — Migration is straightforward, minimal blockers expected
- 🟠 **Feasible** — Migration is achievable with moderate effort, some areas need attention
- 🔴 **Complex** — Significant refactoring or re-architecture required before or during migration

---

## Framework-Only API Reference

APIs that exist only in .NET Framework and require migration to alternatives:

| Framework-Only API | Migration Path | Effort |
|-------------------|---------------|--------|
| `System.Web` (HttpContext, HttpRequest, etc.) | ASP.NET Core `Microsoft.AspNetCore.Http` | High — pervasive changes |
| `System.Web.Mvc` | ASP.NET Core MVC (`Microsoft.AspNetCore.Mvc`) | Medium — API similarity but different pipeline |
| `System.Web.Http` (Web API 2) | ASP.NET Core Controllers or Minimal APIs | Medium — similar patterns |
| `System.ServiceModel` (WCF client) | CoreWCF, gRPC, or `System.ServiceModel` client package | Medium-High |
| WCF Service hosting | CoreWCF or rewrite as gRPC / REST | High |
| `System.Runtime.Remoting` | gRPC, REST, or named pipes | High — complete rewrite |
| `System.EnterpriseServices` (COM+) | Rewrite with modern patterns | High |
| `Global.asax` | `Startup.cs` / `Program.cs` middleware | Medium |
| `System.Web.Security.FormsAuthentication` | ASP.NET Core Identity / Cookie auth | Medium |
| `System.Configuration.ConfigurationManager` | `IConfiguration` / `IOptions<T>` | Low-Medium |
| `System.Drawing` (GDI+) | `System.Drawing.Common` (Windows-only) or `SkiaSharp` / `ImageSharp` | Medium |
| `System.DirectoryServices` | `System.DirectoryServices` (available but limited on Linux) or `Novell.Directory.Ldap` | Low-Medium |
| ASMX Web Services | REST APIs or gRPC | High |
| `System.Transactions.TransactionScope` | Available in .NET 6+ (Windows-only for distributed) | Low (local) / High (distributed) |

---

## Known Framework-Only NuGet Packages

Packages that are .NET Framework-only and need replacement:

| Package | Replacement | Notes |
|---------|-------------|-------|
| `Microsoft.AspNet.WebApi` | `Microsoft.AspNetCore.Mvc` | Complete rewrite of controllers |
| `Microsoft.AspNet.Mvc` | `Microsoft.AspNetCore.Mvc` | View engine changes |
| `Microsoft.AspNet.SignalR` | `Microsoft.AspNetCore.SignalR` | API changes |
| `Microsoft.Owin` | ASP.NET Core middleware | Different pipeline model |
| `EntityFramework` (EF6) | `Microsoft.EntityFrameworkCore` | Migration tool available but manual review needed |
| `Unity` (DI container) | `Microsoft.Extensions.DependencyInjection` or keep Unity with Unity.Microsoft.DependencyInjection | Adapter available |
| `Autofac` (older versions) | `Autofac.Extensions.DependencyInjection` (6.0+) | Modern versions support .NET Core |
| `log4net` | `Serilog`, `NLog`, or `Microsoft.Extensions.Logging` | log4net works on .NET Core but consider modernizing |
| `Newtonsoft.Json` | `System.Text.Json` (optional) | Newtonsoft works on .NET Core, migration optional |
| `Microsoft.ReportingServices` | Alternative reporting libraries | No direct .NET Core equivalent |
| `Crystal Reports` | Alternative reporting | No .NET Core support |

---

## Windows-Only Dependency Severity Guide

### Blocking (🔴) — No cross-platform alternative

- **P/Invoke to custom native DLLs** — requires rewriting native code or maintaining Windows-only deployment
- **COM Interop** — no Linux equivalent, requires complete rewrite
- **WPF / WinForms UI** — desktop-only, no migration path to cross-platform (consider MAUI for new development)
- **MSMQ** — replace with RabbitMQ, Azure Service Bus, or other cross-platform broker
- **Windows Auth (Negotiate/NTLM) as sole auth mechanism** — requires adding alternative auth for non-Windows environments

### Workaround Available (🟠) — Can be replaced

- **Registry access** (`Microsoft.Win32.Registry`) — move to configuration files or environment variables
- **EventLog** (`System.Diagnostics.EventLog`) — replace with structured logging (Serilog, NLog)
- **Windows Services** (`ServiceBase`) — migrate to `BackgroundService` / `IHostedService` with `Microsoft.Extensions.Hosting`
- **Performance Counters** — replace with `System.Diagnostics.Metrics` or Prometheus metrics
- **Windows Certificate Store** — use file-based certs or Azure Key Vault
- **Named Pipes** (Windows-specific usage) — available cross-platform in .NET Core but verify usage patterns
