# 07-1 Boris Cherny Workflow
> How the creator of Claude Code completed 259 PRs in 30 days.

## Learning Objectives
- Understand Boris Cherny's "parallel session" strategy and how it eliminates human wait time
- Master how CLAUDE.md works as "institutional memory"
- Learn what to do at each node of the "Explore → Plan → Implement → Commit" four-step cycle
- Understand why choosing a stronger model is a worthwhile investment in overall efficiency

---

## Main Content

### 1. Background: A Surprisingly Ordinary Story

Boris Cherny is the creator of Claude Code. He completed 259 PRs in 30 days, with every line of code written by Claude, producing approximately 40,000 lines. When people asked him what "secret" he relied on, his answer surprised everyone: "This approach is surprisingly ordinary."

That statement is worth reflecting on. The 259 PRs were not achieved through some magic incantation, but through the continuous stacking of three things: reducing the risk of each change, eliminating human wait time, and continuously accumulating quality knowledge. All three can be systematized, and Boris's 13 core techniques are the concrete expression of that system.

This section focuses on the 7 most important techniques, starting with "why do it this way" before discussing "how to do it."

---

### 2. Technique One: Stick with Opus (or the Strongest Available Model)

**Why:** The cost of fixing errors far exceeds compute costs. When Claude generates an incorrect implementation using a weaker model, you need to spend time: reading the code, identifying the problem, re-prompting, waiting for output again, and verifying again. The time spent in this loop is 10 to 20 times the price difference between Opus and Sonnet. The API costs saved by using a cheaper model get paid back tenfold in time costs during the debug loop.

**How:** For any work that will go through code review, default to the strongest available model. Only consider downgrading when you are certain the task is exploratory and low-risk. Don't skimp on results.

---

### 3. Technique Two: The Parallel 5+5 Pattern

**Why:** Humans are idle while waiting for Claude's response. If you only run one session at a time, your effective work rate is limited by Claude's generation speed, not your judgment speed. Parallelization shifts the human bottleneck from "waiting" to "judging," which is the correct use of human time.

**How:** Run 5 local sessions (local terminals) simultaneously plus 5 to 10 remote sessions (e.g., GitHub Actions or cloud environments). Use macOS system notifications to coordinate switching — when Claude completes a task, a desktop notification pops up, you switch over to review, confirm everything looks good, start the next task, then switch back to other sessions. This keeps your attention always on "wherever human judgment is most needed," rather than watching a cursor blink.

---

### 4. Technique Three: Keep Each PR Focused

**Why:** Large PRs are the enemy of quality. A PR containing 800 lines of changes spanning 5 features cannot be deeply reviewed, is hard to roll back if something goes wrong after merging, and makes it difficult to pinpoint which change caused a bug. Small PRs make every decision clearly visible.

**How:** Enforce two rules: each PR contains only one logical change, and stays under 200 lines. If a feature requires more lines, split it into multiple PRs and merge them in dependency order. This discipline may look like "extra work" in the short term, but it speeds up the entire development cycle because every node is safer.

---

### 5. Technique Four: CLAUDE.md Institutional Memory

**Why:** Every time Claude makes the same mistake, you waste a debug cycle. If the mistake is "preventable" and you haven't written down the prevention rule, you are accepting a repeatable cost that could have been avoided. CLAUDE.md is the mechanism that ensures each mistake only happens once.

**How:** Every time Claude makes a mistake, immediately add a prevention rule to CLAUDE.md after fixing it. Boris's CLAUDE.md is about 2,500 tokens, containing content that the entire team continuously iterates through code review. This is not a document written by one person — it is a knowledge base that grows from real pitfalls.

---

### 6. Technique Five: Explore → Plan → Implement → Commit

**Why:** Jumping straight to implementation is the most common efficiency trap. Without exploration, Claude may make incorrect assumptions about the existing codebase structure. Without a plan, directional issues arise during implementation, requiring you to start over. This four-step cycle front-loads the thinking cost, making the implementation process more linear.

**How:**
- **Explore**: Have Claude read relevant files and understand the existing architecture without making any changes.
- **Plan**: Switch to Plan Mode (`claude --plan`), have Claude list implementation steps, and review and confirm the direction.
- **Implement**: Switch back to Normal Mode and execute according to the plan.
- **Commit**: Use `/commit` or standard git flow to create a PR, keeping the PR focused.

---

### 7. Technique Six: Give Claude a Way to Verify

**Why:** After Claude generates code, if there is no feedback mechanism, it cannot determine whether the output is correct. A verification method is the trigger for Claude's self-correction. Without verification, Claude will assume its output is correct, even when it is wrong.

**How:** Provide at least one verification method: a test script (run and check output), a screenshot (visual verification), or an expected output example (comparison verification). Tell Claude: "After finishing, run this test. If it fails, fix it yourself until it passes." This is the highest-leverage practice in the entire system because it minimizes the human cost of the correction loop.

---

### 8. Technique Seven: The Setup Is Surprisingly Simple

**Why:** Many people assume Boris has a complex custom system to achieve this level of output. This assumption is wrong — and dangerous — because it makes people feel "I'm not ready yet," delaying when they start taking action.

**How:** Boris says his configuration is "vanilla" — no complex custom hooks, no sophisticated agent architecture. The core is just three things: CLAUDE.md (institutional memory) + good prompts (clear task descriptions) + verification methods (letting Claude self-check). You can start using these three things today.

---

### 9. Why This Approach Works

Boris's approach works not because of any magic technique, but because it simultaneously solves three independent efficiency bottlenecks:

**Reducing the risk of each change (small PRs):** Each PR's scope is small enough to be reviewed quickly and thoroughly. Errors are contained within a small scope, and rollback costs are low.

**Eliminating wait time (parallel sessions):** Human attention is always on meaningful work, not waiting for Claude's cursor. Productivity is no longer limited by the response speed of a single session.

**Continuously accumulating quality (CLAUDE.md institutional memory):** Every debug result is converted into a prevention rule, and the system makes fewer mistakes over time. This is a continuous improvement flywheel, not a static configuration.

---

### 4. Comparing the Boris Method vs. the OpenAI Method

Boris's method is not the only high-output workflow. The OpenAI Codex team produced 1 million lines and 1,500 PRs in 5 months with zero hand-written code. Comparing the two methods reveals best practices at different scales:

| Aspect | Boris Cherny | OpenAI Codex Team |
|--------|-------------|-------------------|
| Human role | Write prompts + review every PR | Design environment + specify intent |
| Code output | Every line written by Claude | Every line written by Codex |
| Review | Human reviews every PR | Agents review each other, humans intervene selectively |
| Merge strategy | Small PRs (under 200 lines) + strict review | Merge fast, fixing is cheaper than waiting |
| Cleanup | CLAUDE.md continuous iteration | GC Agent automatically opens refactoring PRs |
| Throughput | ~8 PRs per person per day (259/30) | 3.5 PRs per person per day |
| Core difference | Personal workflow, low barrier to entry | Team system design, requires infrastructure investment |

The two are not contradictory. Boris's method is better suited for individuals or small teams getting started quickly — you can begin using it today. The OpenAI method is better suited for teams with the resources to invest in harness infrastructure — requiring weeks to months of setup.

---

## Teaching Example

### Scenario: Developing a "User Tag Management" Feature Using the Boris Workflow

Suppose you are developing a SaaS platform and need to add a "User Tag Management" feature that allows administrators to tag users, query user lists by specific tags, and remove tags.

**Preparation Phase (Before Starting)**

Your CLAUDE.md already records several rules, such as: "All database queries must use parameterized queries, no string concatenation" and "API endpoints must go through the existing `authMiddleware` for authentication." These are rules added after previous mistakes.

**Step One: Explore**

You open a new Claude Code session and provide this prompt:

"Please read `src/models/User.ts`, `src/routes/users.ts`, `src/middleware/auth.ts`, and understand the existing user data structure and routing patterns. Do not make any changes — just describe the structure you observe."

Claude reports: The user model currently has three fields — `id`, `email`, `role`. Routes use Express, and middleware already has `authMiddleware` wrapping. There is no existing tag system.

**Step Two: Plan**

Switch to Plan Mode:

"Based on the structure you observed, list all the steps needed to implement the 'User Tag Management' feature. The feature includes: tagging users, querying users by specific tags, and removing tags."

Claude lists the plan:
1. Add a `Tag` data model (`id`, `name`, `userId`)
2. Add a database migration
3. Add three API endpoints (POST `/tags`, GET `/tags/:tag`, DELETE `/tags/:id`)
4. Apply `authMiddleware` to all endpoints
5. Add corresponding unit tests

You review the plan, confirm the direction is correct, and add one note: "All queries need pagination using the existing `paginateQuery` utility function."

**Step Three: Open Parallel Sessions**

You notice this feature can be split into two independent PRs:
- PR 1: Data model + migration (no dependencies, can merge first)
- PR 2: API endpoints + tests (depends on PR 1)

You open two sessions. Session A handles PR 1, Session B stands by (you give it an exploration task first, then start implementation after PR 1 is merged).

Your macOS notifications are configured so Session A will send a notification when it finishes.

**Step Four: Give Claude a Way to Verify**

In Session A's prompt, you add: "After finishing, run `npm run migrate:test` and `npm test -- --testPathPattern=tag`. If any tests fail, fix them yourself until all pass."

Session A starts working. You switch to doing other things.

A notification pops up. You come back to check Session A's output: all tests pass, and PR 1's changes are `src/models/Tag.ts` (43 lines) and `migrations/001_add_tags.sql` (12 lines), totaling 55 lines. This meets the under-200-lines rule.

**Step Five: Review and Merge**

You quickly review the 55-line change: the data model structure is correct, the migration has a rollback script, and there is no string concatenation in queries. Review takes 5 minutes, and you merge PR 1.

**Step Six: CLAUDE.md Update**

During review, you notice Claude did not add a `createdAt` field to the Tag model. Although the task spec did not mention it, you know all data tables should have this field. You add a rule to CLAUDE.md: "All new data models must include `createdAt` and `updatedAt` fields using the `timestamp with time zone` type."

Next time Claude creates a new model, this rule will automatically take effect.

**Result**

This feature took approximately 3 hours from exploration to merging both PRs, of which your active work time (thinking, reviewing, decision-making) was about 45 minutes. The rest of the time you were handling other tasks, because parallel sessions reduced your wait time to nearly zero.

This is the everyday face of the Boris Workflow: not magic, but a system of doing the right things properly.

---

## Key Takeaways
- The extra cost of Opus is far less than the time cost consumed by debug loops — using the strongest model for production work is the rational choice
- Parallel sessions shift the human bottleneck from "waiting" to "judging," the most direct way to increase effective work rate
- Each PR should contain only one logical change and stay under 200 lines, making review, merging, and rollback all safer
- CLAUDE.md is institutional memory grown from real pitfalls — every mistake should be converted into a prevention rule
- Giving Claude verification methods (tests, screenshots, expected output) is the single highest-leverage practice in the entire system

---

## Self-Assessment
- If you were to start using the Boris Workflow today, how many rules in your current CLAUDE.md are extracted from real pitfalls? Which ones are just rules that "felt like they should be included"?
- How many lines was your last PR? If it exceeded 200 lines, which part could be split out into an independent PR?
- When Claude is working in a session, what are you doing? If the answer is "waiting," which tasks could you parallelize to eliminate that wait?
