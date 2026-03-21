# Data Platform Toolkit

A Claude Code plugin providing specialized agents and skills for data platform engineering — covering Trino query optimization, Java testing standards, and Kubernetes deployment best practices.

## What's Included

### Skills

| Skill | Description |
|---|---|
| `/trino-optimizer` | Enforces partition filtering, explicit joins, and predicate pushdown for Trino queries on Iceberg tables |
| `/java-testing` | Enforces JUnit 5, Mockito mocking, and strict null-safety assertions |
| `/k8s-deployer` | Enforces resource limits, readiness probes, and liveness probes on all K8s deployments |

### Agents

| Agent | Skills | Description |
|---|---|---|
| `@data-engineer` | `trino-optimizer`, `k8s-deployer` | Data pipeline architect focused on Spark, Iceberg, and Trino at terabyte scale |
| `@java-engineer` | `java-testing`, `k8s-deployer` | JVM backend developer enforcing strict testing and deployment standards |

## Installation

### Option 1: Install from GitHub (recommended)

Inside Claude Code, run:

```
/plugin marketplace add Nonymus10/data-platform-toolkit
/plugin install data-platform-toolkit@Nonymus10-data-platform-toolkit
```

### Option 2: Install from a local clone

Clone the repo and add it as a local marketplace:

```bash
git clone https://github.com/Nonymus10/data-platform-toolkit.git
cd data-platform-toolkit
```

Then inside Claude Code:

```
/plugin marketplace add .
/plugin install data-platform-toolkit@Nonymus10-data-platform-toolkit
```

### Option 3: Try it without installing

Load the plugin for a single session using the `--plugin-dir` flag:

```bash
claude --plugin-dir ./plugins/data-platform-toolkit
```

## Usage

Once installed, you can:

- **Invoke a skill directly** — type `/trino-optimizer`, `/java-testing`, or `/k8s-deployer` to apply its rules to your current task.
- **Invoke an agent** — type `@data-engineer` or `@java-engineer` to get a specialized persona that combines multiple skills.
- **Reload after changes** — if you modify any skill or agent files, run `/reload-plugins` inside Claude Code to pick up the changes.

## Project Structure

```
data-platform-toolkit/
├── .claude-plugin/
│   └── marketplace.json      # Marketplace manifest
├── plugins/
│   └── data-platform-toolkit/
│       ├── .claude-plugin/
│       │   └── plugin.json   # Plugin metadata
│       ├── agents/
│       │   ├── data-engineer.md
│       │   └── java-engineer.md
│       └── skills/
│           ├── trino-optimizer/SKILL.md
│           ├── java-testing/SKILL.md
│           └── k8s-deployer/SKILL.md
├── CLAUDE.md
└── README.md
```
