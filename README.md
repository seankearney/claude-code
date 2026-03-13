# Claude Code Plugins

A collection of [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugins by Sean Kearney.

## Plugins

### dotnet-service-review

A plugin that conducts formal health reviews of .NET microservice repositories using three specialist agents working in parallel.

Navigate to any .NET repo and run:

```
/dotnet-service-review:dotnet-service-review
```

Three agents launch in parallel, analyze the repo, and produce a `SERVICE-REVIEW.md` report.

**Agents:**

| Agent | Analyzes | Scores |
|-------|----------|--------|
| **dotnet-architect** | Architecture, data flow, integration points | — |
| **dotnet-code-reviewer** | SOLID, .NET practices, security, vulnerabilities | Code Quality, .NET Practices, Security |
| **dotnet-test-analyst** | Build, tests, dependencies, documentation | Testing, Maintainability |

**Individual skills** are also available standalone:

```
/dotnet-service-review:architect       # Architecture & data flow diagram
/dotnet-service-review:code-review     # Code quality, .NET practices, security
/dotnet-service-review:test-analysis   # Build, tests, testing trophy, maintainability
```

See [plugins/dotnet-service-review/README.md](plugins/dotnet-service-review/README.md) for full documentation including scoring, MCP server integration, and report details.

## Installation

```
/plugin marketplace add https://github.com/seankearney/claude-code
/plugin install dotnet-service-review@my-plugins
```

## Repository Structure

```
.claude-plugin/
  marketplace.json          # Plugin marketplace manifest
plugins/
  dotnet-service-review/
    .claude-plugin/
      plugin.json           # Plugin metadata
    agents/                 # Specialist agent definitions
    skills/                 # Skill definitions with knowledge files
    README.md               # Detailed plugin documentation
```

## License

MIT
