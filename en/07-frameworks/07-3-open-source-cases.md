# 07-3 Open-Source Harness Cases
> Harness designs from three top open-source projects — and practices you can steal directly.

## Learning Objectives
- Understand why analyzing others' Harnesses is the most effective shortcut to building your own system
- Grasp the core design highlights and portable elements of three open-source cases
- Learn to compare design choices across different Harnesses using seven key concepts
- Be able to extract a minimum viable combination from multiple cases to build a Harness that fits your needs

---

## Main Content

### 1. Why Study Others' Harnesses

Before building your own Claude Code workflow, spending time studying battle-tested open-source Harnesses offers the highest return on investment. There are three reasons:

**First, they are the crystallization of real mistakes.** Every rule in every open-source Harness exists because of an actual error or inefficiency. These design decisions are lessons paid for with time — and you can stand on those hard-won experiences for free.

**Second, breadth of design patterns.** One person can hardly think of every possible architectural choice in a short time. By looking at three different Harnesses, you see three different perspectives on the same problems, broadening your design thinking.

**Third, resist the urge to reinvent the wheel.** Many people spend excessive time on "designing architecture" when building a Harness, while overlooking the fact that "filling in content based on your own mistake log" is what actually makes a Harness effective.

But here is an important caveat: **Directly copying someone else's Harness is an anti-pattern.** Others' rules reflect their mistakes, their tech stack, and their work habits. If you copy 100 rules but 70 of them are irrelevant to your situation, you are just adding noise to your context window.

The correct approach is: **Extract design patterns, then fill in your own hard-won lessons.**

---

### 2. Case One: claude-code-harness

**Source:** github.com/Chachamaru127/claude-code-harness

**Core Design: Simplified Entry Points + Hard Interception**

This project solves a very common problem: as the number of Skills grows, the system becomes hard to use. Its approach is to consolidate 42 old Skills into 5 verb-based Skills:

- `/setup`: Initialize environment and dependencies
- `/plan`: Analyze problems and design solutions
- `/work`: Execute implementation tasks
- `/review`: Review code quality
- `/release`: Prepare and execute releases

These 5 verbs cover the complete development cycle. Users do not need to memorize the semantics of 42 commands — they only need to know "which phase am I in?"

**Most Notable Design: 9 TypeScript Runtime Guardrails (R01-R09)**

This is the most unique aspect of claude-code-harness. Most Harnesses use text rules in CLAUDE.md to constrain Claude's behavior, but text rules have a fundamental problem: Claude can understand rules, but during long conversations or complex tasks, it may "forget" or "bypass" text rules.

TypeScript Guardrails are not prompt warnings — they are code-level interceptions on the execution path. When Claude attempts a prohibited operation, the Guardrail intercepts at the code level, forces an abort, and suggests the correct approach. This upgrades from "reminding Claude to follow rules" to "making violations technically impossible."

The 9 Guardrails cover common high-risk operations: direct production environment operations, merges that skip testing, unsafe environment variable handling, and more.

**4-Perspective Parallel Review**

`/review` is not a single-perspective audit — it simultaneously reviews from four angles:

1. **Functional correctness**: Does the code implement the expected behavior?
2. **Security**: Are there common vulnerabilities such as the OWASP Top 10?
3. **Performance**: Are there obvious performance issues (N+1 queries, unnecessary blocking operations, etc.)?
4. **Maintainability**: Does the code conform to team standards? Is it easy to understand and modify?

Running these four perspectives in parallel means that "functionality looks fine" will not cause the security review to be skipped, and "security is fine" will not cause performance to be overlooked.

**Practices you can steal directly:**
- TypeScript Guardrails for hard interception — elevate the most important rules from text to code
- The 4-perspective Review structure ensures every review covers functionality, security, performance, and maintainability
- Simplify a complex system's entry points to 5 verbs, lowering the barrier to use

---

### 3. Case Two: everything-claude-code

**Source:** github.com/affaan-m/everything-claude-code

**Core Design: Automatic Learning + Scaled Isolation**

This project's scale far exceeds a typical personal Harness: 100+ Skills, a performance optimization system with 28 Subagents. The problem it solves is: **How to make the Harness itself continuously evolve, rather than relying on manual maintenance.**

**Most Notable Design: The learn → evolve Learning Loop**

`/learn-eval` is an automatic pattern extraction tool. It analyzes recent work logs, error logs, and review feedback to extract recurring patterns — whether common error patterns, efficient solution patterns, or repeatedly requested code structures.

`/evolve` transforms the patterns extracted by `/learn-eval` into new Skills. This means the Harness is not static — it learns from each work session and automatically expands its capabilities.

The essence of this loop is: **Automatically distill hard-won lessons into reusable rules, without human intervention.**

**28-Subagent Isolation Architecture**

In large projects, different languages, frameworks, and modules have different working rules. If all rules live in the same context, they interfere with each other.

The 28-Subagent design means each Subagent only knows the rules it needs to know. A React Subagent is not distracted by Python typing rules; a security audit Subagent is not distracted by UI development rules. This isolation makes each Subagent more focused and reliable in its own domain.

**On-Demand Language Rule Installation**

Rules for different tech stacks are not preloaded — they are installed only when needed. When your task switches to a different language or framework, the relevant rule set is loaded into context at that point. This avoids the context pollution caused by "all rules being present at once."

**Practices you can steal directly:**
- The learn → evolve learning loop — automatically distill mistakes into rules
- Subagent isolation architecture — rules for different domains do not share context
- On-demand language rule installation — prevent irrelevant rules from consuming context space

---

### 4. Case Three: OpenAI Codex Internal Team

| Item | Description |
|------|-------------|
| Source | OpenAI official article "Harness Engineering" (2026-02) |
| Scale | 5 months, ~1 million lines of code, ~1,500 PRs, 3→7 engineers |
| Constraint | Zero handwritten code — every line generated by Codex (GPT-5) |
| Core Principle | Humans steer, Agents execute |
| Speed | 3.5 PRs per person per day, ~10x traditional speed |

**Design Highlights:**

1. **AGENTS.md as a Map**: ~100 lines, pointing to a structured knowledge base under `docs/` (design documents, execution plans, product specs, reference materials), achieving progressive disclosure

2. **Strict Architectural Layering**: Types → Config → Repo → Service → Runtime → UI, with dependency direction strictly validated and mechanically enforced by custom linters and structural tests

3. **Agent-to-Agent Review**: Agent self-review → request review from another Agent → iterate until satisfactory ("Ralph Wiggum Loop") — humans do not need to review every time

4. **Garbage Collection Agent**: Automatically scans for drift in the background and opens refactoring PRs, replacing the initial practice of 20% manual cleanup time every Friday

5. **Observability Integration**: Each worktree has an independent log / metrics / trace stack; Agents can query with LogQL and PromQL, making prompts like "ensure startup completes within 800ms" executable

**Practices you can steal:**
1. Use AGENTS.md / CLAUDE.md as a map with `docs/` storing structured knowledge — already taught in Module 02-1 of this course
2. Write custom linter error messages as Agent repair instructions — low cost, high reward (see 03-1 Advanced Techniques)
3. Garbage Collection Agent that automatically opens refactoring PRs — replaces manual audits

---

### 5. Seven Key Concepts × Case Cross-Reference Table

The most effective way to understand the four cases is to compare their design choices using the same conceptual framework:

| Concept | claude-code-harness | everything-claude-code | gstack | OpenAI Codex |
|---------|-------------------|----------------------|--------|-------------|
| Context Architecture | Progressive Skill loading | On-demand language rules | CLAUDE.md as index | AGENTS.md 100 lines + docs/ |
| Architectural Constraints | 9 TS guardrails | Multi-language Rules enforcement | `/careful` warnings | 6-layer architecture + custom linter |
| Self-Verification Loop | `/harness-work all` | checkpoint evals | `/qa` automated testing | Agent-to-Agent review |
| Context Isolation | 3-Agent division of labor | 28-Subagent firewall | 18 independent Skills | Independent environment per worktree |
| Entropy Management | agent-trace.jsonl | `/learn-eval` → `/evolve` | `/document-release` | GC Agent auto-refactoring |
| Removability | v3 middleware removable | Selective installation | Skill files independent | Not explicitly discussed |
| Agent Readability | — | — | — | Linter errors = repair instructions |

This table reveals several interesting design differences:

**Different levels of architectural constraint strength.** claude-code-harness chose the strongest form of constraint (code-level interception), while gstack chose the lightest (warning prompts). This reflects different design philosophies: the former believes rules must be enforced, the latter believes engineers should be able to bypass rules when necessary.

**Different levels of entropy management automation.** everything-claude-code's learn → evolve loop is automatic; gstack's `/document-release` is manual. Automation comes at the cost of complexity; manual approaches come at the cost of depending on human discipline.

**Different granularity of removability.** everything-claude-code allows selective installation of individual language rule packages, claude-code-harness's v3 middleware can be removed as a whole, and each of gstack's Skill files is independent. All three maintain removability, but at different granularities.

---

## Teaching Example

### Scenario: Building a "Minimum Harness Combination" from Scratch

Suppose you are a full-stack developer, independently building a SaaS product with TypeScript + Next.js, handling 10 to 15 feature and bug fix tasks per week. You have never built a systematic Harness before — each time you just give Claude ad-hoc prompts. You decide to build your first Harness.

Steal one core design from each of the three cases. Here is the recommended "minimum Harness combination":

**Steal from claude-code-harness: 5 Verb Entry Points**

This is the highest priority to adopt, because it solves the problem of "I have a Harness but can't remember how to use it." Your minimum version only needs three Skills:

- `/plan`: Analyze the task, list steps, identify risks
- `/work`: Execute implementation with verification conditions
- `/review`: Review from 4 perspectives (functionality, security, performance, maintainability)

You do not need `/setup` and `/release` for now, since your environment is already configured and your release process is familiar. Start with the three highest-frequency entry points.

**Steal from everything-claude-code: Manual Learning Loop**

You do not yet have the resources to build a full automatic `/learn-eval` → `/evolve` loop, but you can build a manual version:

Add a section in CLAUDE.md called "This Week's Lessons Learned." After each task, spend 5 minutes answering:
1. What Claude errors or inefficiencies did I encounter this time?
2. How can I prevent this next time through prompts or Skills?

Every Friday, distill the contents of the "This Week's Lessons Learned" section into new CLAUDE.md rules or updates to existing Skills. This is a manual learn → evolve loop, but it builds the same habit: **Every mistake must be converted into a system improvement.**

**Steal from gstack: /office-hours Probing Mechanism**

Add a mandatory step to your `/plan` Skill: before listing implementation steps, you must first answer three questions (a simplified version of office-hours):

1. Where is this task most likely to fail?
2. Is there a simpler solution than my first instinct?
3. If this task is done wrong, what is the cost of correction?

These three questions are lighter than gstack's six questions, but they preserve the core function: **Force yourself to uncover the most important hidden assumptions before starting work.**

**Your Minimum Harness Structure**

```
CLAUDE.md              ← Core rules + This Week's Lessons Learned section
.claude/
  commands/
    plan.md            ← 3 probing questions + step breakdown
    work.md            ← Task execution + verification conditions
    review.md          ← 4-perspective review checklist
```

This structure totals roughly 150 to 200 lines, covers the highest-frequency use cases, and does not require a complex Agent architecture.

**Why This Combination Works**

These three designs complement each other without overlap:
- The 5 verb entry points solve "how to enter the workflow"
- The learning loop solves "how to make the system continuously improve"
- The probing mechanism solves "how to avoid moving fast in the wrong direction"

More importantly, every element in this combination comes from real-case validation, not theoretical design. You are starting from a foundation of others' hard-won experience, not groping from scratch.

**Expansion Path**

After working with this minimum combination for four to six weeks, you should have your own list of lessons learned. At that point, you can decide on the next expansion based on specific pain points:

- If you find Claude repeatedly making the same type of security errors, consider adopting the TypeScript Guardrail concept from claude-code-harness — elevate rules from text to code
- If your tasks start involving multiple tech stacks, consider adopting the on-demand rule package loading from everything-claude-code
- If your task scale grows to require multi-role collaboration, consider adopting the role division design from gstack

The core logic of this path is: **Start with the minimum combination to build habits, then decide expansion direction based on real friction encountered.** Do not preemptively import complexity you do not need before you have actually hit that particular wall.

---

## Key Takeaways
- The correct way to analyze open-source Harnesses is to "extract design patterns and fill in your own lessons learned" — not to copy directly
- claude-code-harness's most unique contribution is TypeScript Runtime Guardrails, elevating rules from text to code-level hard interception
- everything-claude-code's learn → evolve loop solves the fundamental problem of Harnesses requiring manual maintenance
- The seven-concept framework (context architecture, architectural constraints, self-verification loop, context isolation, entropy management, removability) can be used to systematically compare design differences between any two Harnesses
- Minimum Harness combination principle: Start from the highest-frequency use cases, let real mistakes drive expansion, and do not preemptively introduce complexity

---

## Self-Assessment
- Using the seven-concept framework, examine your current workflow (or CLAUDE.md) — which concept is completely missing? Has this gap caused you actual pain points?
- If you were to build a minimum Harness today, what are the three most important rules from your "This Week's Lessons Learned" list?
- What is the core difference between TypeScript Runtime Guardrails and CLAUDE.md text rules? In your current work, which rule is worth upgrading to hard interception?
