# dotnet-service-review

A Claude Code plugin that conducts formal health reviews of .NET microservice repositories using three specialist agents working in parallel.

## Usage

Navigate to any .NET repo and run:

```
/dotnet-service-review:dotnet-service-review
```

Three agents launch in parallel, analyze the repo, and produce `SERVICE-REVIEW.md`.

### Scoping a Subdirectory

For monorepos or multi-service solutions, scope the review to a specific service:

```
/dotnet-service-review:dotnet-service-review src/PaymentService
```

The orchestrator will automatically warn if the target is too large (>5 projects or >500 source files) and ask you to narrow scope.

### Run Agents Individually

Each agent is also available standalone:

```
/dotnet-service-review:architect       # Architecture & data flow diagram
/dotnet-service-review:code-review     # Code quality, .NET practices, security
/dotnet-service-review:test-analysis   # Build, tests, testing trophy, maintainability
```

No configuration required. The repo you're in IS the service.

### Providing Institutional Context

Create a `SERVICE-REVIEW-CONTEXT.md` in the repo root to give the agents domain knowledge they can't discover from code alone:

```markdown
# Service Review Context

## Domain
This service is part of the Batch Processing domain. Related repos: core, listener-rabbitmq.

## Known Issues
- The GenerateACHFile class uses virtual methods (WriteBatchHeader, WriteDetail, etc.)
  to support custom ACH DLLs via inheritance. Refactoring must preserve this contract.
- CheckMarx has flagged SQL injection in GetRecordsCount repeatedly; prior fixes
  addressed symptoms, not root cause.

## Team
Single active contributor. Bus factor = 1. Onboarding documentation is sparse.

## Architecture Notes
- Shares dbo.PAYMENTS table with core — no domain API boundary exists.
- OFAC screening is documented in wiki as happening here, but actually lives in core.
```

This file is **optional**. When present, its contents are passed to all three agents to inform their analysis. It's particularly valuable for:

- Context that lives in people's heads, not in code
- Known issues or technical debt that should be verified, not rediscovered
- Cross-repo relationships that aren't visible from this repo alone
- Historical context (past security scans, incident patterns, failed refactors)

## How It Works

The orchestrator launches three specialist agents **in parallel**:

| Agent | Analyzes | Scores | Key MCP Tool |
|-------|----------|--------|-------------|
| **dotnet-architect** | Architecture, data flow, integration points, wiki docs | — | GitNexus |
| **dotnet-code-reviewer** | SOLID, .NET practices, security, vulnerabilities | Code Quality, .NET Practices, Security | RoslynMCP |
| **dotnet-test-analyst** | Build, tests, dependencies, documentation | Testing, Maintainability | Bash |

The orchestrator collects their outputs, computes the overall health score, synthesizes recommendations, and writes the final report.

## MCP Servers

Works with **zero MCP servers**. Significantly deeper with them.

| MCP Server | Used By | What It Adds |
|---|---|---|
| **GitNexus** | architect | Cross-repo dependency mapping, blast radius analysis |
| **RoslynMCP** | code-reviewer | Cyclomatic complexity, compiler diagnostics, symbol analysis |
| **Documentation** (Confluence, Notion) | architect | Wiki context for architectural decisions |

## Report Includes

- Overall health score with 5-category breakdown
- Architecture overview with Mermaid data flow diagram
- Key integration points table
- Build & test results
- Testing trophy (current vs. recommended)
- Detailed findings per category with file:line references
- Top 3 prioritized recommendations

## Scoring

Traffic-light (Green/Orange/Red) across 5 categories:

```
GREEN  = 0 Red AND <= 2 Orange
ORANGE = 1-2 Red OR 3+ Orange
RED    = 3+ Red OR any Critical failure
```

## Plugin Structure

```
dotnet-service-review/
├── .claude-plugin/
│   └── plugin.json
├── agents/
│   ├── dotnet-service-review.md      # Orchestrator
│   ├── dotnet-architect.md           # Architecture specialist
│   ├── dotnet-code-reviewer.md       # Code quality specialist
│   └── dotnet-test-analyst.md        # Test & build specialist
├── skills/
│   ├── dotnet-service-review/
│   │   ├── SKILL.md                  # Full review (orchestrated)
│   │   └── knowledge/               # 8 shared knowledge files
│   ├── architect/
│   │   └── SKILL.md                  # Standalone architecture analysis
│   ├── code-review/
│   │   └── SKILL.md                  # Standalone code review
│   └── test-analysis/
│       └── SKILL.md                  # Standalone test analysis
└── README.md
```

## License

MIT
