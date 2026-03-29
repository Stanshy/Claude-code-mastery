# 02-1 CLAUDE.md Complete Guide
> CLAUDE.md is the most important configuration file in Claude Code. It ensures Claude understands your project rules from the very first second, rather than guessing from scratch every time.

## Learning Objectives
- Understand why CLAUDE.md has the highest priority among all configuration files
- Master the four file locations, their applicable scenarios, and scopes
- Learn to judge what content should be included and what should be excluded
- Be able to create an effective CLAUDE.md for a real project within 10 minutes

---

## Main Content

### 1. Why CLAUDE.md Is the Most Important Configuration

CLAUDE.md is the instruction file that Claude Code **automatically loads** at the beginning of every session. It determines how Claude understands your project. Without CLAUDE.md, Claude guesses your preferences from scratch every time; with CLAUDE.md, Claude knows your rules from the very first second.

This is not ordinary documentation. It is an operations manual written for Claude, directly influencing every decision Claude makes: which command to run tests with, how to handle errors, and what code style trade-offs to make.

Boris Cherny, the creator of Claude Code, said: "Every time Claude does something wrong, add it to CLAUDE.md so it never repeats the same mistake." His own CLAUDE.md is only about 2,500 tokens, continuously iterated by the entire team during code review. This statement reveals the essence of CLAUDE.md: it is a **living document** that grows from mistakes, not a static description written once and forgotten.

### 2. File Locations and Scopes

CLAUDE.md can exist in multiple locations, each with a different scope:

| Location | Scope | Purpose |
|----------|-------|---------|
| `./CLAUDE.md` (project root) | Single project | Committed to git, shared with the team |
| `~/.claude/CLAUDE.md` | All projects | Personal preferences (e.g., language, style) |
| Subdirectory `packages/api/CLAUDE.md` | Specific submodule | Independent rules for each subsystem in a monorepo |
| `/etc/claude-code/CLAUDE.md` | Organization level | Unified deployment by administrators |

Claude automatically loads all CLAUDE.md files from the working directory up to the root directory, as well as `~/.claude/CLAUDE.md`. Multiple CLAUDE.md files are **merged together** and do not override each other.

Practical example: when you are working in the `packages/api/` directory, Claude simultaneously loads `./CLAUDE.md` (root directory rules), `packages/api/CLAUDE.md` (API module-specific rules), and `~/.claude/CLAUDE.md` (your personal preferences). All three layers of rules take effect at the same time, so you don't need to repeat global settings in every subdirectory.

### 3. What to Include vs. What to Exclude

This is where beginners most commonly make mistakes. The core principle of CLAUDE.md is: **only write what Claude cannot guess on its own**.

| Should Include | Should Exclude |
|----------------|----------------|
| Bash commands Claude cannot guess (e.g., `pnpm run dev:local`) | Content Claude can infer from package.json |
| Code style that differs from defaults (e.g., tab vs space) | Standard language conventions (e.g., Python's PEP8) |
| Test commands and preferences (e.g., `jest --watch`) | Detailed API documentation (use `@` imports instead) |
| Project-specific architectural decisions | Information that changes frequently |
| Mandatory rules (emphasized with `IMPORTANT` or `YOU MUST`) | Obvious practices |

Writing "This is a Node.js project" is a waste of context. Claude sees `package.json` and naturally knows. But writing "Use ES module syntax, do not use CommonJS" is valuable, because Node.js supports both syntaxes and Claude has no way to determine your choice without additional hints.

Mandatory rules should use explicit language. "Preferably use" and "YOU MUST use" have vastly different effects on Claude's behavior. For non-negotiable rules, mark them with `IMPORTANT:` or `YOU MUST`.

### 4. `/init` Quick Start

Run `/init` in your project directory, and Claude will automatically analyze the codebase and generate an initial version of CLAUDE.md. This is the fastest way to get started — you'll have a usable draft within 5 minutes.

However, note that the version produced by `/init` usually **requires manual refinement**. Auto-generated content tends to include everything it can describe, including many things Claude can actually infer on its own. The first thing to do after generation is remove the redundancy and keep only the rules that only you know.

### 5. `@` Import Syntax

CLAUDE.md supports `@` references to external files, avoiding overload from cramming everything into a single file:

```markdown
@README.md
@docs/api-spec.md
@.claude/rules/testing.md
```

This lets you keep CLAUDE.md itself concise while importing deeper documentation on demand. An API specification file might be hundreds of lines long and is not suitable for direct inclusion in CLAUDE.md, but with an `@` import, Claude can still reference it when needed.

### 6. Size Control Strategies

The longer CLAUDE.md gets, the higher the probability that Claude ignores some rules. This is not a flaw of Claude — it is an engineering reality of the context window.

Several mainstream strategies:

- **Boris Cherny's method**: Maintain approximately 2,500 tokens (about 200 lines), co-maintained by the entire team, iterated during each code review
- **Minimalist school**: Keep it under 60 lines, including only universally applicable rules
- **David Chu's integrated approach**: Start with Boris's method to build the habit of "adding mistakes to CLAUDE.md," then once rules mature, "graduate" frequently triggered rules to Hooks / Skills / Agents, returning CLAUDE.md to a lean state

Practical advice: **Keep CLAUDE.md under 200 lines**. Beyond that number, consider splitting content into the `.claude/rules/` directory, referencing via `@`, or converting to automated Hooks.

### 8. Designing for Agent Readability (Not Humans)

OpenAI discovered a counterintuitive principle through their practice with Codex: a codebase's documentation and structure should be optimized first and foremost for Agent readability.

Specific practices:
- **Keep all decisions in the repo**: Architecture decisions, design documents, and product specifications should all live in the `docs/` directory, not in external tools (Google Docs, Notion, Slack). From the Agent's perspective, knowledge not in the repo is equivalent to nonexistent.
- **Prefer "boring" technology**: Choose composable technologies with stable APIs that are well-represented in training data. Agents understand mainstream technologies more reliably than obscure frameworks. In some cases, having the Agent re-implement a subset of functionality is cheaper than dealing with opaque third-party packages.
- **Customize linter error messages**: Linter error descriptions should be written as fix instructions that the Agent can directly execute, rather than merely describing the problem.

The implication for CLAUDE.md is: CLAUDE.md is not just a set of rules for Claude — it is the entry point for the Agent to understand the entire project. Any knowledge the Agent cannot discover from the repo is equivalent to nonexistent.

### 9. The Division of Labor Between the Memory System and CLAUDE.md

Claude Code has two memory mechanisms, and confusing them is a common misunderstanding:

| System | CLAUDE.md | Auto Memory |
|--------|-----------|-------------|
| Who writes it | You | Claude |
| Content | Rules and instructions | Learnings and patterns |
| When loaded | Every session | Every session |
| Purpose | Rules that always apply | Project-specific insights |

Auto Memory is located at `~/.claude/projects/<project>/memory/`, and Claude writes to it automatically during work. For example, after Claude discovers your test file naming convention, it records it for reference in subsequent sessions.

You do not need to manage Auto Memory manually. Your responsibility is to maintain the rules in CLAUDE.md; Claude's responsibility is to accumulate observations in Auto Memory. The two systems are complementary — do not rely on Auto Memory to remember rules that belong in CLAUDE.md, because you cannot directly control the content of Auto Memory.

---

## Hands-On Example: Writing CLAUDE.md for the AI Dev Assistant
> Continuing the main case study: in the AI Dev Assistant project, this is the first step in the "architecture capability track" and the foundation for all subsequent operations in the entire AI Dev Assistant project.

**Goal**: Write an effective CLAUDE.md for the ai-dev-assistant project, ensuring all subsequent Claude sessions automatically follow project rules

**Prerequisites**: Completed 01-2 (project skeleton setup), the `ai-dev-assistant/` directory already exists and contains `package.json`

**Steps**:

1. Navigate to the project directory and launch Claude Code:
   ```bash
   cd ai-dev-assistant
   claude
   ```

2. Run `/init` within Claude Code to let Claude automatically analyze the project structure and generate an initial CLAUDE.md:
   ```
   /init
   ```

3. Claude will analyze `package.json`, `tsconfig.json`, the directory structure, etc., and generate a draft. After it completes, open `CLAUDE.md` in your editor.

4. Remove everything from the draft that Claude can infer on its own. Refine according to the following template (this template is the official version for the ai-dev-assistant project):

   ```markdown
   # AI Dev Assistant

   CLI tool that automates code review, bug fixing, PR analysis, and test generation.

   ## Tech Stack
   - Node.js 18.x (TypeScript 5.x)
   - Testing: Jest 29.x
   - Linting: ESLint 8.x + Prettier 3.x

   ## Core Rules
   - Use ES module syntax (import/export), do not use CommonJS (require)
   - All functions must have explicit TypeScript type annotations
   - Must pass `npm test` and `npm run lint` before committing
   - Use custom Error classes for error handling, do not use string throw

   ## Common Commands
   - Build: `npm run build`
   - Test: `npm test`
   - Single test: `npx jest --testPathPattern=<filename>`
   - Lint: `npm run lint`
   - Format: `npx prettier --write .`

   ## Project Structure
   - `src/` — Core logic
   - `tests/` — Jest tests
   - `scripts/` — Automation scripts
   - `.claude/` — Claude Code configuration
   ```

5. After saving `CLAUDE.md`, clear the current session in Claude Code and test whether the rules take effect:
   ```
   /clear
   ```
   Then enter a simple task, for example: "Create a utils.ts file in src/ and add a formatDate function." Observe whether Claude uses ES module syntax and adds TypeScript type annotations.

6. If Claude violates a rule (for example, using `require` instead of `import`), immediately add this violation scenario to the core rules in CLAUDE.md, and explain **why** this rule exists:
   ```markdown
   ## Core Rules
   - IMPORTANT: Must use ES module syntax (import/export).
     This project's package.json has "type": "module" set, and using require will cause runtime errors.
   ```

**Expected Results**:
- A `CLAUDE.md` appears in the project root, no longer than 200 lines
- All subsequent Claude sessions automatically follow the tech stack rules (ES modules, TypeScript type annotations)
- Claude uses `npm test` to run tests instead of guessing other commands
- `CLAUDE.md` is committed to git, taking effect immediately when team members clone the repo

**Common Mistakes**:
- **CLAUDE.md is too long (over 300 lines)**: Claude starts ignoring rules toward the end, and the most important rules are most likely to be missed. Solution: trim to under 200 lines, and use `@` references or move detailed explanations to the `.claude/rules/` subdirectory.
- **Including content Claude can infer on its own** (e.g., "This is a Node.js project," "Uses TypeScript"): Wastes limited context space. Solution: only write rules that Claude cannot correctly guess without additional hints.
- **Writing rules in suggestive tone** (e.g., "Preferably use ES modules"): Claude will treat it as an optional suggestion rather than a hard requirement. Solution: prefix all mandatory rules with `IMPORTANT:` or `YOU MUST`.

---

## Key Takeaways
- CLAUDE.md is automatically loaded at the start of every session and is the most direct mechanism for influencing Claude's behavior
- Only write what Claude cannot guess on its own to avoid wasting context; mark mandatory rules with `IMPORTANT` or `YOU MUST`
- CLAUDE.md files in four locations are merged together: root directory (team-shared), subdirectory (submodule-specific), `~/.claude/` (personal preferences), `/etc/claude-code/` (organization deployment)
- Use `/init` to quickly generate a draft, then manually refine and remove redundancy
- Keep CLAUDE.md under 200 lines; beyond that, split it out or convert to Hooks

## Self-Check
- Does your CLAUDE.md contain anything Claude can infer from `package.json` or the directory structure on its own? If so, delete it.
- When Claude does something wrong in your project, do you add the mistake and the correct approach to CLAUDE.md?
- How many lines is your CLAUDE.md right now? If it exceeds 200 lines, which content can be moved to separate files referenced via `@`?
