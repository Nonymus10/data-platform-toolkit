# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

This is a **Claude Code plugin** called `data-platform-toolkit`. It is not a traditional application — there is no build step, no tests, and no runtime. It consists of Markdown-based skill and agent definitions that Claude Code loads at runtime.

## Architecture

This repo follows the same plugin packaging model as the Claude Code reference plugins:

- **Marketplace manifest**: `.claude-plugin/marketplace.json`
- **Plugin package root**: `plugins/data-platform-toolkit/`
- **Plugin metadata**: `plugins/data-platform-toolkit/.claude-plugin/plugin.json`
- **Auto-discovered components**:
  - Agents in `plugins/data-platform-toolkit/agents/*.md`
  - Skills in `plugins/data-platform-toolkit/skills/*/SKILL.md`

### Skill → Agent Mapping

| Agent | Skills |
|---|---|
| `data-engineer` | `trino-optimizer`, `k8s-deployer` |
| `java-engineer` | `java-testing`, `k8s-deployer` |

`k8s-deployer` is shared across both agents — changes to it affect both personas.

## Editing Guidelines

- Skill rules should be strict and enforceable (e.g., "MUST", "never", "reject if"). Vague guidance gets ignored.
- Agent system prompts should specify *when* to apply each skill, not just list them.
- Do not list `agents` or `skills` arrays in `plugin.json`; Claude discovers components from folder structure at install/runtime.
- Adding a new skill requires creating `plugins/data-platform-toolkit/skills/<name>/SKILL.md` with valid frontmatter.
- Adding a new agent requires creating `plugins/data-platform-toolkit/agents/<name>.md` with valid frontmatter and `skills[]`.
