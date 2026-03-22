# Data Platform Toolkit

A Claude Code plugin providing specialized agents and skills for data platform engineering — covering Trino query optimization, Java testing standards, Kubernetes deployment best practices, full-service API flow mapping, security architecture, and surgical codebase transformation with OWASP security and system design patterns.

## What's Included

### Agents

| Agent | Skills | Description |
|---|---|---|
| `@data-engineer` | `trino-optimizer` | Data pipeline architect focused on Spark, Iceberg, and Trino at terabyte scale. Enforces partition filtering, explicit joins, and predicate pushdown on every query. |
| `@java-engineer` | `java-testing`, `k8s-deployer` | JVM backend developer enforcing strict testing and deployment standards. Blocks PRs without tests and deployments without health probes. |
| `@cartographer` | `cartographer` | Meticulous reverse-engineering agent that maps complete codebase internals. Produces an exhaustive `docs/service-map.md` tracing every API endpoint, call chain, middleware, DB schema, and architecture quality. |
| `@service-surgeon` | `service-surgeon`, `java-testing`, `k8s-deployer` | Senior architect that executes any codebase transformation — features, refactoring, security hardening, performance optimization, migrations. Consumes cartographer output, plans with blast radius analysis, enforces factory pattern, OWASP Top 10, and zero-downtime migrations. |
| `@java-security-expert` | `java-security-expert`, `java-testing` | Java security architect specializing in Spring Boot and modular design. Covers JWT/OAuth2 (RS256), RBAC/ABAC, OWASP Top 10 remediation, cryptography, secrets management, audit logging, rate limiting, and CORS hardening. |

### Skills

| Skill | Description |
|---|---|
| `trino-optimizer` | Enforces partition filtering, explicit joins, and predicate pushdown for Trino queries on Iceberg tables |
| `java-testing` | Enforces JUnit 5, Mockito mocking, and strict null-safety assertions |
| `k8s-deployer` | Enforces resource limits, readiness probes, and liveness probes on all K8s deployments |
| `cartographer` | Maps complete service flows — traces every endpoint through its full call chain with structured, machine-parseable output |
| `service-surgeon` | Consumes cartographer output and executes surgical codebase transformations with factory pattern, OWASP enforcement, and system design patterns |
| `java-security-expert` | Expert guidance on Java app security — Spring Security, JWT/OAuth2, modular architecture, factory pattern, and OWASP Top 10 defense |

## Installation

### Option 1: Install from GitHub (recommended)

**Step 1 — Add the marketplace** to Claude Code:

```
/plugin marketplace add Nonymus10/data-platform-toolkit
```

This registers the Synapse marketplace from GitHub. You can also pin a specific version:

```
/plugin marketplace add Nonymus10/data-platform-toolkit#v1.0.0
```

**Step 2 — Install the plugin** from the marketplace:

```
/plugin install data-platform-toolkit@Synapse
```

This installs the plugin to your **user** scope (available across all projects). To install to a specific scope:

```bash
# Project scope — shared via version control (.claude/settings.json)
claude plugin install data-platform-toolkit@Synapse --scope project

# Local scope — project-only, gitignored
claude plugin install data-platform-toolkit@Synapse --scope local
```

**Step 3 — Verify** the plugin is installed:

```
/plugin
```

Navigate to the **Installed** tab — you should see `data-platform-toolkit` listed.

### Option 2: Install from a local clone

**Step 1 — Clone the repo:**

```bash
git clone https://github.com/Nonymus10/data-platform-toolkit.git
cd data-platform-toolkit
```

**Step 2 — Add as a local marketplace** inside Claude Code:

```
/plugin marketplace add .
```

**Step 3 — Install the plugin:**

```
/plugin install data-platform-toolkit@Synapse
```

### Option 3: Try it without installing

Load the plugin for a single session using the `--plugin-dir` flag:

```bash
claude --plugin-dir ./plugins/Synapse
```

This does not persist across sessions — the plugin is only available for the current session.

### Managing the Plugin

```bash
# List all marketplaces
/plugin marketplace list

# Update the marketplace to pull latest changes
/plugin marketplace update Synapse

# Disable the plugin (without uninstalling)
/plugin disable data-platform-toolkit@Synapse

# Re-enable the plugin
/plugin enable data-platform-toolkit@Synapse

# Uninstall the plugin
/plugin uninstall data-platform-toolkit@Synapse

# Remove the marketplace entirely
/plugin marketplace remove Synapse

# Reload plugins after making changes to agent/skill files
/reload-plugins
```

## Usage

### Using Agents

Agents are specialized personas that combine multiple skills. Invoke them with the `@` prefix using the plugin namespace:

```
@data-platform-toolkit:data-engineer
@data-platform-toolkit:java-engineer
@data-platform-toolkit:cartographer
@data-platform-toolkit:service-surgeon
@data-platform-toolkit:java-security-expert
```

**Example prompts:**

```
@data-platform-toolkit:cartographer map this service and produce a service-map.md

@data-platform-toolkit:service-surgeon refactor the auth module to use factory pattern

@data-platform-toolkit:java-security-expert review this service for OWASP compliance

@data-platform-toolkit:data-engineer optimize this Trino query for partition pruning

@data-platform-toolkit:java-engineer add unit tests for the UserService class
```

### Using Skills

Skills are reusable knowledge domains that can be invoked directly with the plugin namespace:

```
/data-platform-toolkit:trino-optimizer
/data-platform-toolkit:java-testing
/data-platform-toolkit:k8s-deployer
/data-platform-toolkit:cartographer
/data-platform-toolkit:service-surgeon
/data-platform-toolkit:java-security-expert
```

### Recommended Workflow

1. **Map first** — Run `@data-platform-toolkit:cartographer` on a service to produce a complete `docs/service-map.md`.
2. **Then transform** — Run `@data-platform-toolkit:service-surgeon` to execute changes with full knowledge of the codebase.
3. **Secure** — Run `@data-platform-toolkit:java-security-expert` to review for OWASP compliance and security architecture.

## Project Structure

```
data-platform-toolkit/
├── .claude-plugin/
│   └── marketplace.json            # Marketplace manifest
├── plugins/
│   └── Synapse/
│       ├── .claude-plugin/
│       │   └── plugin.json         # Plugin metadata
│       ├── agents/
│       │   ├── cartographer.md
│       │   ├── data-engineer.md
│       │   ├── java-engineer.md
│       │   ├── java-security-expert.md
│       │   └── service-surgeon.md
│       └── skills/
│           ├── cartographer/SKILL.md
│           ├── java-security-expert/SKILL.md
│           ├── java-testing/SKILL.md
│           ├── k8s-deployer/SKILL.md
│           ├── service-surgeon/SKILL.md
│           └── trino-optimizer/SKILL.md
├── CLAUDE.md
└── README.md
```
