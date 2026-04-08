# Claude Code Mastery: From Beginner to Expert

> Stanshy | 2026-03

---

## Overview

This course is built on David Chu's Five-Layer Evolution Model, systematically guiding readers from Claude Code fundamentals all the way to Harness Engineering and autonomous Agent systems.

All chapters revolve around a single running case project (AI Dev Assistant), ensuring every concept has actionable, hands-on examples.

---

## Course Structure

| Module | Evolution Layer | Topic | Chapters |
|--------|----------------|-------|----------|
| 1 | Layer 1: Conversational | Introduction to Claude Code | 2 |
| 2 | Layer 2: Configured | Configuration | 3 |
| 3 | Layer 3: Automated | Hooks / Skills / MCP | 4 |
| 4 | Layer 4: Orchestrated | Subagents / Context / Parallel Dev | 3 |
| 5 | Layer 5: Autonomous | Agent Teams / Headless / SDK | 3 |
| 6 | Methodology | Harness Engineering | 2 |
| 7 | Case Studies | Real-World Frameworks | 4 |
| 8 | Reference | Cheatsheet, Resource Index, Disaster Recovery | 3 |

---

## Chapter Index

### Module 1: Introduction to Claude Code

| Chapter | Title | Summary |
|---------|-------|---------|
| [01-1](01-intro/01-1-what-is-claude-code.md) | What is Claude Code | Agentic CLI definition, Agent Loop, tool comparison |
| [01-2](01-intro/01-2-installation.md) | Installation & First Success | Three installation methods, authentication, project scaffolding |

### Module 2: Configuration

| Chapter | Title | Summary |
|---------|-------|---------|
| [02-1](02-config/02-1-claude-md-guide.md) | CLAUDE.md Complete Guide | Boris Cherny philosophy, writing standards, size control |
| [02-2](02-config/02-2-settings-permissions.md) | Settings & Permissions | settings.json layers, five permission modes, allow/deny syntax |
| [02-3](02-config/02-3-harness-preview.md) | Agent = Model + Harness | Harness core concept preview, with vs without Harness comparison |

### Module 3: Automation

| Chapter | Title | Summary |
|---------|-------|---------|
| [03-1](03-automation/03-1-hooks-basics.md) | Hooks Basics | Deterministic vs advisory, 6 core events, configuration format |
| [03-2](03-automation/03-2-hooks-advanced.md) | Advanced Hooks & Validator Pattern | Stop Hook Validator Pattern, four handler types |
| [03-3](03-automation/03-3-skills.md) | Skills System | On-demand loading, SKILL.md format, dynamic content injection |
| [03-4](03-automation/03-4-mcp-servers.md) | MCP Servers | External tool protocol, GitHub MCP, Tool Search |

### Module 4: Orchestration

| Chapter | Title | Summary |
|---------|-------|---------|
| [04-1](04-orchestration/04-1-subagents.md) | Custom Subagents | Independent context, role definition, three invocation methods |
| [04-2](04-orchestration/04-2-context-management.md) | Context Management Strategies | 8 management techniques, Document & Clear method |
| [04-3](04-orchestration/04-3-worktree.md) | Parallel Development & Worktree | Git Worktree, Boris's 5+5 parallel mode |

### Module 5: Autonomous

| Chapter | Title | Summary |
|---------|-------|---------|
| [05-1](05-autonomous/05-1-agent-teams.md) | Agent Teams | Team Lead + Teammates collaboration architecture |
| [05-2](05-autonomous/05-2-headless-cicd.md) | Headless Mode & CI/CD Integration | Non-interactive mode, GitHub Actions automation |
| [05-3](05-autonomous/05-3-agent-sdk.md) | Agent SDK | Python/TypeScript SDK programmatic control |

### Module 6: Harness Engineering

| Chapter | Title | Summary |
|---------|-------|---------|
| [06-1](06-harness/06-1-seven-principles.md) | Seven Design Principles | Context architecture, constraints, self-validation, isolation, entropy, modularity, agent readability |
| [06-2](06-harness/06-2-build-from-zero.md) | Build a Harness from Scratch | Pitfall-driven six-step process, three anti-patterns, full evolution example |

### Module 7: Real-World Frameworks

| Chapter | Title | Summary |
|---------|-------|---------|
| [07-1](07-frameworks/07-1-boris-cherny.md) | Boris Cherny Workflow | 259 PRs in 30 days, 5+5 parallel mode, CLAUDE.md institutional memory |
| [07-2](07-frameworks/07-2-gstack.md) | gstack Role System | YC president's 18-skill engineering team simulation |
| [07-3](07-frameworks/07-3-open-source-cases.md) | Open-Source Harness Cases | Three top project design analyses, seven-concept cross-reference |
| [07-4](07-frameworks/07-4-david-chu.md) | David Chu Evolution Framework | Five-layer model, three graduation paths, Boris vs RPI integration |

### Module 8: Reference

| Chapter | Title | Summary |
|---------|-------|---------|
| [08-1](08-reference/08-1-cheatsheet.md) | Cheatsheet | Commands, shortcuts, Hook events, permission syntax, troubleshooting |
| [08-2](08-reference/08-2-resources.md) | Resource Index | Curated 37 resources, Top 10 must-bookmarks |
| [08-3](08-reference/08-3-disaster-recovery.md) | Disaster Recovery & Debug Guide | 5 fault modes, debugging workflows, --resume complete usage |

---

## Recommended Reading Order

| Background | Suggested Path |
|------------|---------------|
| Never used Claude Code | Module 1 → 2 → 3 → 4 → 5 |
| Know basics, want to level up | Module 3 → 4 → 6 → 7 |
| Want to learn Harness Engineering | Module 6 → 7 → 3 |
| Limited time, just 3 chapters | 01-1 → 02-1 → 03-2 |
