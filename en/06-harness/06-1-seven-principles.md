# 06-1 Seven Design Principles
> Tools alone do not make a system — the seven design principles help you decide when to use which tool, how much, and when to remove it.

## Learning Objectives
- Understand why the system surrounding the model, not the model itself, is the bottleneck for Agent reliability
- Master the definitions of the seven design principles and the specific mechanisms each maps to in Claude Code
- Be able to identify what "doing it right" and "doing it wrong" looks like for each principle
- Be able to use the seven principles to review and evaluate whether your Harness is complete

## Content

### 1. Why Design Principles Are Needed

So far, you have learned about CLAUDE.md, Hooks, Skills, Subagents, MCP, and Worktree. These are tools. But tools alone do not make a system — you need a set of design principles to decide when to use which tool, how much, and when to remove it. This is Harness Engineering.

The word "Harness" comes from motorsport engineering: a seatbelt is not the engine, but without a seatbelt, no matter how powerful the engine, the driver cannot push to the limit. The Harness in Claude Code works the same way — components like CLAUDE.md, Hooks, and Skills do not make Claude smarter; they allow its capabilities to be reliably and repeatably guided in the right direction.

**Core thesis**: The reliability bottleneck of an Agent is not in the model, but in the system surrounding the model.

The problems that LangChain was repeatedly criticized for by the community in 2023-2024 were not about GPT-4 being insufficiently powerful, but about design flaws in the framework's context management, tool-calling logic, and error handling. Conversely, Anthropic engineers observed in internal experiments on multiple occasions that by only optimizing the surrounding prompt structure, tool design, and validation loops — without changing the model — task success rates could jump from 40% to over 80%. The model stayed the same; the Harness changed.

This observation gave rise to the seven design principles. Each principle corresponds to a category of real failure modes and has a clear implementation path within Claude Code's mechanisms.

---

### 2. The Seven Principles in Detail

#### Principle 1: Context Architecture

**One-line definition**: Give a map, not an encyclopedia. The Agent looks things up on demand rather than preloading everything.

**Root cause**: A language model's context window is a limited resource. Stuffing all potentially relevant information into the context does not make Claude stronger — it dilutes the truly important information while increasing inference cost and error rates. Claude performs far better in a precise 50-line context than in a noisy 2,000-line context.

**Mapping to Claude Code**:
- `CLAUDE.md`: Serves as an index (< 200 lines), telling Claude "where to find what" rather than putting all content directly inside
- `Skills`: Loaded on demand (`/skill-name`), bringing in specific knowledge blocks only when needed
- `@` imports: Reference specific files on the fly during conversation without preloading

**Doing it right**: CLAUDE.md is 50 lines total, containing a project architecture overview, key rules as bullet points, and reference paths to deeper documentation. When the API specification is needed, Claude reads it via `@docs/api-spec.md`.

**Doing it wrong**: All API documentation, architecture diagrams, ADRs (Architecture Decision Records) from three months ago, and complete specifications are pasted into CLAUDE.md, pushing it to 3,000 lines. Claude has to sift through noise every time to find what it actually needs.

---

#### Principle 2: Architectural Constraints

**One-line definition**: Use tools to enforce rules, not prompts to suggest them.

**Root cause**: A prompt is a suggestion, not a constraint. Instructions like "please do not do X" are often ignored under pressure (for example, when Claude has failed multiple attempts and starts trying shortcuts). Enforcement at the tool layer is the real constraint — a Hook returning `exit 2` physically prevents Claude from continuing.

**Mapping to Claude Code**:
- `Hooks`: PreToolUse intercepts dangerous operations (blocking before tool execution); PostToolUse callbacks validate results
- `settings.json` deny rules: Directly prohibit specific CLI tools or arguments

**Doing it right**: A Hook intercepts `git push --force`. The script detects the `--force` argument and issues `exit 2`, outputting "Force push has been blocked by the Harness. Please use --force-with-lease and explain your reason." Claude receives the blocking message and cannot bypass it.

**Doing it wrong**: CLAUDE.md says "Please do not force push, it is dangerous." Claude follows this under normal circumstances, but after five failed rebase attempts — when it considers force push the last resort — it may still choose to force push. Prompt suggestions fail under pressure.

---

#### Principle 3: Self-Validation Loop

**One-line definition**: Intercept dead loops and enforce validation before exit.

**Root cause**: Claude saying "done" does not mean it is actually done. The model may make an error in the last step, forget to run tests, or introduce new bugs during modifications. Without mandatory exit validation, "Agent completed" simply equals "Agent claims completed."

**Mapping to Claude Code**:
- `Stop Hook Validator`: Before Claude is about to end a conversation, it automatically executes predefined validation scripts (e.g., `npm test && npm run lint`). If validation fails, Claude is forced back into the work loop rather than delivering flawed results.

**Doing it right**: The Stop Hook runs `npm test && npm run lint`. One test case fails, the Hook returns a failure status, Claude sees the specific error output and fixes the issue, then attempts to exit again — this time validation passes.

**Doing it wrong**: Claude modifies three files and says "Task complete, I have updated the relevant logic." You trust it and merge directly. CI runs the tests and finds a type error introduced in the second file. The cost of fixing is your time, not Claude's.

---

#### Principle 4: Context Isolation

**One-line definition**: Use Subagents as firewalls — they do not pollute each other.

**Root cause**: Long conversations accumulate intermediate artifacts — Claude's reasoning process, results of failed attempts, temporary assumptions. These artifacts consume context, interfere with subsequent judgments, and can even cause Claude to make more errors by staying within an "established line of thinking." The solution for complex tasks is not to give Claude a longer context, but to let it work in a clean environment.

**Mapping to Claude Code**:
- `Subagents`: Each Subagent has its own independent context. After investigation or generation work is done, only a concise result is returned to the main conversation
- `Worktree`: Subagents operate in isolated Git worktrees — file-system-level isolation that does not affect the main branch

**Doing it right**: You need to investigate a complex codebase security issue. Delegate to a Security Reviewer Subagent. The Subagent reads fifty files and builds an analysis report in its own independent context. The main conversation receives only a 300-word conclusion, keeping the context clean.

**Doing it wrong**: All work is done in the main conversation. Claude investigates the security issue, modifies code, generates a report, and makes further modifications — all in the same context. By the 30th conversation turn, the context is full of outdated intermediate reasoning, and Claude starts contradicting itself.

---

#### Principle 5: Entropy Management

**One-line definition**: The longer a system runs, the messier it gets — active maintenance is required.

**Root cause**: Adding a rule after every pitfall is correct. But if you only add and never remove, after three months you have 30 Hooks — some conflicting with each other, some addressing problems already handled natively by newer models, some never triggered at all. Entropy increase in systems is inevitable — the Harness needs periodic active maintenance.

**Mapping to Claude Code**:
- `Auto Memory`: Claude Code automatically records important learnings from conversations, preventing the same issues from recurring
- Pitfall-to-rule closed loop: Each real pitfall maps to a specific rule (Hook / Skill / CLAUDE.md entry)
- Periodic CLAUDE.md audits: Monthly or quarterly reviews to remove outdated entries and consolidate duplicate rules

**Doing it right**: Every Friday, spend 15 minutes reviewing the pitfall log. This week Claude submitted PRs twice without tests — distill this into a PreToolUse Hook that intercepts `git commit` when no corresponding test file exists. At the same time, discover that a rule added three weeks ago — "do not use `var`" — is already covered by an ESLint rule. Delete it.

**Doing it wrong**: Add rules when pitfalls occur, never delete. Three months later, CLAUDE.md has 150 lines, 60 of which are outdated or duplicated. The Hooks directory has 12 scripts — two have never been triggered, and one pair has contradictory logic. The system gets heavier, and problems become harder to track.

**Advanced approach (OpenAI pattern)**: Instead of relying on manual periodic audits, deploy automated Agents running in the background:

- **Garbage Collection Agent**: Periodically scans for codebase drift (inconsistent naming, duplicated logic, architecture violations) and automatically opens refactoring PRs. Most can be reviewed and auto-merged within a minute. The OpenAI team initially spent 20% of their time every Friday manually cleaning up "AI slop," then replaced this with automated Agents.
- **Doc-gardening Agent**: Scans for outdated documentation, checks consistency between docs and code, and automatically submits fix PRs.

This is the ultimate form of entropy management: human taste only needs to be captured once (written as rules), then continuously and automatically enforced on every line of code.

---

#### Principle 6: Removability

**One-line definition**: The Harness must be modular — stronger models can remove some components.

**Root cause**: Model capabilities are evolving rapidly. Much of the Harness that Claude 3 needs may already be handled natively by Claude 4. If your Harness is a monolithic whole, you cannot remove unnecessary parts after a model upgrade, and the system grows ever heavier with increasing maintenance costs.

**Mapping to Claude Code**:
- Each Hook is an independent shell script — deleting a single file does not affect other Hooks
- Each Skill is an independent Markdown file — removing it causes Claude to fall back to default behavior
- Each Agent is an independent prompt file — disabling it does not affect the main Harness

**Doing it right**: Monthly audit — "Has this Hook been triggered in the last 30 days?" "Can the newer version of Claude handle the capability this Skill provides on its own?" Discover that `auto-add-semicolons.sh` has never triggered since Prettier was introduced. Delete it. The system is a bit lighter, and nothing else is affected.

**Doing it wrong**: All constraint logic, all Agent configurations, and all special rules are written into a single massive `settings.json` with interdependent logic. To remove one rule, you need to read the entire file to assess the impact — the risk is too high, so you end up not touching anything. The system can only grow; it can never shrink.

---

#### Principle 7: Agent Legibility

**One-line definition**: Optimize the codebase for Agent readability first, not humans.

**Root cause**: From the Agent's perspective, anything it cannot access in its context at execution time is equivalent to nonexistent. Knowledge that lives in Google Docs, Slack conversations, or someone's head is inaccessible to the Agent. Just as a new employee will not know about a Slack discussion from three months ago, the Agent will not know about decisions that were never written into the repo.

In their practice of producing 1 million lines of code and 1,500 PRs with Codex over 5 months, OpenAI developed this into a core design philosophy: the repo itself is the single source of truth. All architecture decisions, design documents, and product specifications must exist within the repo, not in external systems.

**Mapping to Claude Code**:
- `CLAUDE.md` as the entry-point map — all architecture decisions must be discoverable by the Agent from within the repo
- Documentation stored in structured form within the repo (`docs/`, `.claude/rules/`), linked via `@` references rather than placed in Notion or Google Docs
- Custom linter error messages designed as fix instructions the Agent can directly execute, rather than merely describing the problem

**Doing it right**: When the linter reports an error, it outputs "This function is missing type annotations. Please add TypeScript types to the parameters and return value. Example: `function add(a: number, b: number): number`." The Agent reads the error message and immediately knows how to fix it. The team's architecture decisions are documented in `docs/design-docs/`, and CLAUDE.md points to them with `@`.

**Doing it wrong**: The linter error only says `error TS7006: Parameter implicitly has an 'any' type`. The Agent needs to search documentation separately to understand the fix, reducing success rates. The team's architecture decisions are in Confluence, completely inaccessible to the Agent.

---

### 3. Seven Principles x Claude Code Reference Table

| Principle | Claude Code Mechanism | Chapter Reference |
|-----------|----------------------|-------------------|
| Context Architecture | CLAUDE.md + Skills + `@` imports | 02-1, 03-3 |
| Architectural Constraints | Hooks + deny rules | 03-1, 02-2 |
| Self-Validation Loop | Stop Hook Validator | 03-2 |
| Context Isolation | Subagents + Worktree | 04-1, 04-3 |
| Entropy Management | Auto Memory + pitfall closed loop + GC Agent | 02-1, 04-2 |
| Removability | Modular file structure | Entire book |
| Agent Legibility | CLAUDE.md + structured docs + linter message design | 02-1, 03-1 |

How to use this table: When building or reviewing a Harness, each row serves as a checkpoint. The question is not "am I using this mechanism," but "am I using this mechanism to correctly achieve the goal of the corresponding principle."

---

## Hands-On Example: Auditing the AI Dev Assistant with the Seven Principles

> Continuing the main case study: In the AI Dev Assistant project, you have completed all configurations from Modules 1-5. Now use the seven principles as a review framework to evaluate whether the entire Harness is truly effective, not merely "present."

**Objective**: Apply the seven principles one by one to the AI Dev Assistant's existing Harness, identify gaps, and document improvement items.

**Prerequisites**:
- All implementations from Modules 1-5 completed (CLAUDE.md, Hooks, Skills, Subagents, Worktree)
- The local `ai-dev-assistant` project can successfully run `npm test` and `npm run lint`
- Node.js 18.x, TypeScript 5.x, Jest 29.x, ESLint 8.x properly installed

**Steps**:

1. List all current Harness components for the AI Dev Assistant and create an inventory:
   ```bash
   find .claude -type f | sort
   cat CLAUDE.md | wc -l
   ```
   Record the output. This is your "current state snapshot" for later comparison.

2. Create a seven-principle checklist (can be a plain text file `harness-audit.md`), with three statuses for each principle: Complete / Partially Complete / Missing. Do not score yet — continue checking and fill in afterward.

3. Run the Principle 1 check — is Context Architecture in place:
   - Is the CLAUDE.md line count below 200? (`wc -l CLAUDE.md`)
   - Is the content of CLAUDE.md index-style ("detailed spec at `docs/api-rules.md`") or encyclopedia-style (all specs pasted directly)?
   - Are there Skills covering "on-demand loading" knowledge blocks?
   Fill in the checklist with your conclusions.

4. Run the Principle 2 check — are Architectural Constraints using the tool layer rather than the prompt layer:
   - List all "please do not..." and "avoid..." entries in CLAUDE.md — these are prompt suggestions, not constraints
   - For each, determine: if Claude violates this rule, are the consequences serious? If so, it should be upgraded to a Hook
   - Do existing Hooks cover the highest-risk operations? (e.g., force push, deleting production data, directly modifying the `main` branch)

5. Run the Principle 3 check — is the Self-Validation Loop working:
   - Does the Stop Validator Hook exist and is it enabled? (`cat .claude/hooks/stop-validator.sh`)
   - Manually trigger once: after completing any task in Claude Code, observe whether `npm test && npm run lint` runs automatically
   - If a test fails, does the Stop Validator actually prevent Claude from finishing?

6. Run the Principle 4 check — is Context Isolation actually being used:
   - For the last three complex tasks (more than 10 files, more than 5 back-and-forth exchanges), did you use Subagents?
   - If not, how large was the main conversation's context at task completion? (observe from Claude Code's token count)
   - Is Worktree correctly configured and used when needed?

7. Run the Principle 5 check — does Entropy Management have a closed-loop mechanism:
   - Is there a pitfall log? (e.g., `.claude/pitfalls.md` or similar file)
   - When was the last time you reviewed the pitfall log? How many days ago?
   - What is the oldest entry in CLAUDE.md? Are there rules that no longer apply?

8. Run the Principle 6 check — can components be independently removed:
   - In the `.claude/hooks/` directory, can each script be deleted individually without affecting the others?
   - In the last 30 days, has each Hook been triggered? (you can add logging to scripts to track this, or observe Claude Code's output log)
   - Is there a Hook whose corresponding issue the latest version of Claude can now handle on its own, making the hard interception no longer necessary?

**Expected outcomes**:
- `harness-audit.md` is completed, with each principle having a clear status rating (Complete / Partially Complete / Missing)
- At least 1-2 specific improvement items identified (e.g., "Principle 2 missing: three 'please do not' rules in CLAUDE.md should be upgraded to Hooks")
- A next-steps action list: each "Partially Complete" or "Missing" item maps to a specific modification plan

**Common mistakes**:

- **Checking only "does it exist" rather than "does it work"**: The Stop Validator script exists, but a syntax error in the script prevents it from executing correctly — this is equivalent to not having one. A Hook exists, but the trigger condition is misconfigured and it has never fired — this is equivalent to not having one. Audits must include functional verification, not just file existence checks.

- **Not performing periodic audits, only reviewing when problems arise**: Entropy increase in the Harness is gradual — a single outdated component will not immediately cause an obvious problem. By the time components conflict with each other, the cost of fixing is far greater than the cost of periodic maintenance. A fixed monthly 15-minute Harness audit is recommended — it is cheaper than spending three hours troubleshooting after something breaks.

---

## Key Takeaways
- The core thesis of Harness Engineering: the reliability bottleneck of an Agent is in the system surrounding the model, not the model itself
- Among the seven principles, Principle 2 (Architectural Constraints) is most commonly overlooked — engineers habitually write suggestions in CLAUDE.md rather than constraints in the Hook layer
- Principle 3 (Self-Validation Loop) is the single mechanism with the highest return on investment — the Stop Validator can dramatically improve delivery quality with almost no additional conversation turns
- Principle 6 (Removability) determines the long-term maintenance cost of the Harness — modular design costs 10% more time upfront but saves over 50% in subsequent maintenance
- The seven principles form a whole: missing any one diminishes the effectiveness of the other six

## Self-Assessment
- When was the last time your CLAUDE.md exceeded 200 lines? What did you do about it? (If you have never audited, that itself is a problem)
- List the rules in your Harness that currently exist as prompt suggestions — which ones have consequences serious enough that they should be upgraded to Hooks?
- If you upgrade Claude to the next major version right now, which of your Hooks or Skills might no longer be necessary? How would you verify this?

---

⬅️ [Previous: Agent SDK](../05-autonomous/05-3-agent-sdk.md) ｜ 📖 [Index](../00-index.md) ｜ ➡️ [Next: Build a Harness from Scratch](06-2-build-from-zero.md)

🌐 [繁體中文版](../../zh/06-Harness-Engineering/06-1-七大設計原則.md)
