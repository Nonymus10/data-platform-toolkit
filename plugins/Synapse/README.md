# data-platform-toolkit plugin package

This folder is the installable Claude Code plugin package.

- Plugin metadata: `.claude-plugin/plugin.json`
- Agents: `agents/*.md`
- Skills: `skills/*/SKILL.md`

Components are auto-discovered by Claude Code at install/runtime.

## Installation

See the [root README](../../README.md) for full installation instructions. Quick start:

```
/plugin marketplace add Nonymus10/data-platform-toolkit
/plugin install data-platform-toolkit@Synapse
```

Or try it for a single session without installing:

```bash
claude --plugin-dir ./plugins/Synapse
```

## Agents

Invoke agents with `@data-platform-toolkit:<agent-name>`.

| Agent | Skills | When to Use |
|---|---|---|
| `@data-platform-toolkit:data-engineer` | `trino-optimizer` | Managing terabyte-scale data pipelines built on Spark, Iceberg, and Trino. Use when writing/reviewing Spark jobs, Trino queries, or deploying data ingestion workers. Enforces partition filtering, explicit joins, and predicate pushdown. |
| `@data-platform-toolkit:java-engineer` | `java-testing`, `k8s-deployer` | Building and shipping production Java services. Use when you need strict JUnit 5 testing with Mockito and null-safety, and Kubernetes deployments with resource limits and health probes. Blocks PRs without tests and deployments without probes. |
| `@data-platform-toolkit:cartographer` | `cartographer` | Reverse-engineering and mapping a codebase. Use when you need a complete understanding of a service before making changes — produces an exhaustive `docs/service-map.md` covering every API endpoint, call chain, middleware, DB schema, error handling, and architecture quality assessment. |
| `@data-platform-toolkit:service-surgeon` | `service-surgeon`, `java-testing`, `k8s-deployer` | Executing codebase transformations (features, refactoring, security hardening, performance optimization, migrations). Use after `@cartographer` has mapped the service. Plans surgery with blast radius analysis, enforces factory pattern, OWASP Top 10, zero-downtime migrations, and system design patterns (strangler fig, circuit breaker, CQRS, saga, outbox). |
| `@data-platform-toolkit:java-security-expert` | `java-security-expert`, `java-testing` | Securing Java/Spring Boot applications. Use for security architecture reviews, implementing authentication/authorization (JWT/OAuth2 with RS256), RBAC/ABAC, OWASP Top 10 remediation, cryptography, secrets management, audit logging, rate limiting, and CORS hardening. Enforces modular security architecture and blocks insecure patterns in code review. |

## Skills

Invoke skills with `/data-platform-toolkit:<skill-name>`.

| Skill | Description |
|---|---|
| `/data-platform-toolkit:trino-optimizer` | Optimizes Trino query performance against Apache Iceberg tables — enforces partition filtering, explicit joins, and predicate pushdown-friendly patterns. |
| `/data-platform-toolkit:java-testing` | Enforces rigorous Java testing standards using JUnit 5, Mockito, and strict null-safety assertions. |
| `/data-platform-toolkit:k8s-deployer` | Enforces Kubernetes deployment best practices including CPU/memory resource limits, readiness probes, and liveness probes. |
| `/data-platform-toolkit:cartographer` | Maps complete service flows with structured, machine-parseable output — traces every API endpoint through its full call chain, DB operations, external calls, and error handling. |
| `/data-platform-toolkit:service-surgeon` | Expert implementation agent that consumes cartographer output and executes surgical codebase transformations with factory pattern, OWASP enforcement, and system design patterns. |
| `/data-platform-toolkit:java-security-expert` | Expert guidance on Java application security — Spring Security architecture, JWT/OAuth2, modular security design, factory pattern, and OWASP Top 10 defense. |
