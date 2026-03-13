---
name: modernization-analysis
description: Assess migration complexity and runtime/deployment profile of the .NET repo in the current directory. Produces a Modernization Readiness score with detailed evidence tables.
context: fork
agent: dotnet-modernization-analyst
---

# Modernization Analysis

Assess the modernization readiness of this repository.

Produce the **Modernization Readiness** section including:
- Migration Complexity signals with evidence (target framework, framework-only APIs, NuGet compatibility, data access, configuration)
- Runtime & Deployment signals with evidence (hosting model, containerization, CI/CD, Windows-only deps)
- Composite Modernization Readiness score

Read your required knowledge files first. Follow your output format exactly.
