---
name: dotnet-service-review
description: Full health review of the .NET repo in the current directory. Launches architect, code reviewer, and test analyst agents in parallel, then assembles a scored report.
context: fork
agent: dotnet-service-review
---

# .NET Service Review

Conduct a full health review of this repository (or a subdirectory within it) by orchestrating three specialist agents.

## Scope

This review is designed for **microservice-sized repos** (≤5 projects, ≤200 source files).

- **Default**: reviews the current working directory
- **Subdirectory**: pass a path to review a specific service within a larger repo (e.g., `src/PaymentService`)
- **Large repos**: the orchestrator will warn if the repo exceeds size thresholds and ask you to confirm or narrow scope

## What Happens

1. Scope is validated (project/file count checked against thresholds)
2. `SERVICE-REVIEW-CONTEXT.md` is read from repo root if present (institutional knowledge)
3. Three agents launch **in parallel** on the scoped directory:
   - **dotnet-architect** — architecture, data flow diagram, integration points
   - **dotnet-code-reviewer** — code quality, .NET practices, security
   - **dotnet-test-analyst** — build, tests, testing trophy, maintainability
4. Results are collected and assembled into a single scored report
5. Report is written to `SERVICE-REVIEW.md` in the repo root

## Recommended MCP Servers

| MCP Server | Used By | What It Provides |
|---|---|---|
| **GitNexus** | dotnet-architect | Cross-repo dependency mapping, blast radius, integration context |
| **RoslynMCP** | dotnet-code-reviewer | Code metrics, compiler diagnostics, symbol analysis |
| **Documentation** (Confluence, Notion) | dotnet-architect | Wiki context for architectural decisions |

All agents work without MCP servers — they enhance depth when available.

## Also Available Individually

Run any agent standalone:
- `/dotnet-service-review:architect` — just the architecture analysis
- `/dotnet-service-review:code-review` — just the code review
- `/dotnet-service-review:test-analysis` — just the build/test analysis
