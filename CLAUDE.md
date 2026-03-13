# Claude Code Plugins

This repository contains Claude Code plugins authored by Sean Kearney.

## Repository Structure

- `.claude-plugin/marketplace.json` — Plugin marketplace manifest listing all plugins in this repo
- `plugins/` — Individual plugin directories

## Plugins

### dotnet-service-review
A Claude Code plugin that performs full .NET service health reviews using three specialist agents (architect, code reviewer, test analyst) working in parallel. Scores code quality, .NET practices, testing, security, and maintainability.

- `plugins/dotnet-service-review/.claude-plugin/plugin.json` — Plugin metadata
- `plugins/dotnet-service-review/agents/` — Agent definitions (dotnet-architect, dotnet-code-reviewer, dotnet-service-review, dotnet-test-analyst)
- `plugins/dotnet-service-review/skills/` — Skill definitions (architect, code-review, dotnet-service-review, test-analysis)
- `plugins/dotnet-service-review/skills/dotnet-service-review/knowledge/` — Reference knowledge files (patterns, rubrics, templates)

## Development Notes

- Plugins follow the Claude Code plugin specification with `plugin.json` manifests
- Skills are defined in `SKILL.md` files within each skill directory
- Agents are defined as markdown files in the `agents/` directory
- Knowledge files provide domain expertise to skills and agents
