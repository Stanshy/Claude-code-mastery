# 06-2 Build a Harness from Scratch
> Let the Agent run bare first, observe what mistakes it makes, then add targeted constraints — every rule that enters the Harness should correspond to a real problem that has already occurred.

## Learning Objectives
- Understand the "pitfall-driven" Harness-building philosophy, and why you should not design preventively
- Master the six-step process from bare runs to a complete Harness, including the specific artifacts produced at each step
- Identify and avoid three common Harness-building anti-patterns
- Complete the final product feature (test generation) for the AI Dev Assistant, and demonstrate the full Harness evolution through this process

## Main Content

### 1. Core Principle: Pitfall-Driven

Most engineers, when first attempting to build an Agent system, instinctively do one thing: design a perfect Harness first, then start using the Agent. This instinct is wrong.

The reason is: you don't know what mistakes Claude will make in your specific context until it actually makes them. Every project has a different tech stack, workflow, and team conventions. Rules copied from someone else's Harness may address pitfalls they encountered but that you will never encounter. Preventive design brings redundant constraints, not protection.

The logic of pitfall-driven development is: **Let the Agent run bare first, observe what mistakes it makes, then add targeted constraints.**

"Running bare" does not mean being reckless. You should still start with low-risk tasks (reading, analyzing, generating drafts) rather than letting Claude run bare in a production environment. But before building a formal Harness, you need real pitfall data, not an imagined list of pitfalls.

**David Chu's Three Graduation Signals** — When the following three signals appear, it's time to promote a pitfall into a formal Harness component:

1. **CLAUDE.md reaches about 50 lines**: This means there is enough project-specific knowledge that needs to be recorded, making it worth investing time to organize it into an indexed structure
2. **Claude stops consistently following a certain rule**: Prompt suggestions start losing effectiveness, indicating that the consequences of violating this rule are severe enough to warrant upgrading it to an enforced constraint at the Hook layer
3. **It's time for a scheduled review**: A proactive monthly audit, rather than waiting for pitfalls to trigger a review

Every rule that enters the Harness should be able to answer this question: "Which real incident does this rule correspond to?" If the answer is "to prevent a problem that might occur," the rule's place in the Harness is questionable.

---

### 2. Six-Step Process

Harness construction is linear — each step builds on the artifacts from the previous step. In the AI Dev Assistant main case study, the following six steps are the path you actually walked through in Modules 1-5. Now we break it down clearly so you can reproduce this process in your next project.

**Step 0: Bare Run + Record Pitfall List**

Let Claude execute several real tasks without any Harness. Your job is to observe and record, not to intervene. Recommended recording format:

```
Date | Task Description | Claude's Incorrect Behavior | Severity (Low/Medium/High)
```

This list is the raw material for all subsequent Harness components. Without this list, every rule you add is a guess.

**Step 1: Create a Minimal CLAUDE.md (50-100 Line Index)**

After a few bare runs, you'll see some recurring problems: Claude doesn't know which test framework this project uses, isn't clear on the API path naming conventions, or isn't familiar with the TypeScript strict mode settings. These are the material for CLAUDE.md.

A 50-100 line CLAUDE.md should contain:
- Project tech stack (Node.js version, TypeScript config, major packages used)
- Directory structure description (what goes in `src/`, `tests/`, `docs/`)
- The most important work conventions (e.g., all API responses must include `requestId`)
- Reference paths to deep files (e.g., "See `docs/api-spec.md` for detailed API specifications")

**Step 2: Add the First Constraint (Most Frequent or Most Severe Error)**

Pick one item from the pitfall list. The criterion is: most severe consequence, or highest frequency. Don't add multiple at once — you need to be able to evaluate the effectiveness of each constraint.

In the AI Dev Assistant, the first Hook is typically intercepting commits without running tests: a PreToolUse Hook that confirms `npm test` passes before `git commit` executes.

**Step 3: Add a Self-Validation Hook (Stop Validator)**

A Stop Validator makes sense for almost all projects, so it's a standard Step 3 rather than being pitfall-dependent. Design principles for Stop Hooks:

- Execution time should be under 60 seconds (otherwise Claude's workflow will be noticeably disrupted)
- Only validate the two most critical things: tests pass, lint has no errors
- On failure, output specific error messages so Claude knows what needs to be fixed

```bash
#!/bin/bash
# .claude/hooks/stop-validator.sh
npm test --silent 2>&1
TEST_EXIT=$?

npm run lint --silent 2>&1
LINT_EXIT=$?

if [ $TEST_EXIT -ne 0 ] || [ $LINT_EXIT -ne 0 ]; then
  echo "Stop Validator: Tests or lint failed, Claude must fix before finishing"
  exit 2
fi

echo "Stop Validator: Passed"
exit 0
```

**Step 4: Split Out the First Subagent (When Context Overload Occurs)**

When you start noticing that certain types of tasks cause the main conversation's context to rapidly expand — such as analyzing an unfamiliar large codebase or systematically reviewing security vulnerabilities — that's the signal to delegate to a Subagent.

The first Subagent is usually an "investigation type": give it a question and read access, let it work in an independent context, and it returns a concise analysis report. In the AI Dev Assistant, this is `agents/security-reviewer.md`.

**Step 5: Establish a Pitfall-to-Rule Closed Loop**

This is the mechanism that keeps the Harness evolving. After each pitfall, complete a full closed loop:

1. Record the pitfall (log)
2. Analyze the root cause (Why did Claude make this mistake?)
3. Decide on the corresponding Harness component (Hook / Skill / CLAUDE.md entry / Agent directive)
4. Implement and test
5. Update the log, mark as "loop closed"

The most common point where the loop breaks is Step 3: the pitfall is recorded, but no one goes back to analyze it and distill it into a rule. This is why a regular review mechanism is needed (e.g., a fixed 15 minutes every Friday).

**Step 6: Periodic Audit, Remove Outdated Components**

Execute once a month. For each Harness component, ask three questions:
- Has this component been triggered/used in the last 30 days?
- Has the latest version of Claude already built-in handling for this issue?
- If this component is removed, what is the worst case? Is it acceptable?

If the answers are "no," "yes," and "acceptable," remove it.

---

### 3. Three Anti-Patterns

The purpose of understanding anti-patterns is not to criticize, but to help you quickly identify and correct your own behavioral patterns when you see them.

| Anti-Pattern | Symptoms | Solution |
|-------------|----------|----------|
| Perfectionism Trap | Spending two weeks designing a Harness before starting to use the Agent; Harness documentation takes longer to write than actual usage time | Run first, build later. Set a "no more than one workday" rule: bare run on day one, start building the first component on day two |
| Constraint Accumulation Syndrome | Only adding, never removing; 30 rules that conflict with each other; Hook directory has 20 scripts and no one knows what each one does | Before adding a new rule, ask: can an old rule be removed because of this new one? Monthly audits are the systematic solution |
| Copy-Paste Syndrome | Directly copying someone else's CLAUDE.md or Hooks; the Harness contains entries where you can't explain what problem they address | Every rule must be able to answer "Which real pitfall does this correspond to?" Entries that can't answer this should be deleted or marked as "pending validation" |

---

## Hands-on Example: Complete Evolution of the AI Dev Assistant Harness (Including Test Generation Feature)

> Main case study continued: In the AI Dev Assistant project, you have completed all implementations from Modules 1-5. This example does two things: reviews the trajectory of the entire Harness evolution, and adds the final product feature — test generation.

**Goal**:
- Confirm that the AI Dev Assistant's Harness has completed the full six-step process
- Add `src/test-gen.ts` (generateTests function), demonstrating the Harness's practical role in new feature development
- Perform a complete Step 6 audit

**Prerequisites**:
- Modules 1-5 completed (CLAUDE.md, settings.json deny rules, Stop Validator Hook, Security Reviewer Agent)
- Node.js 18.x environment, `npm test` runs successfully (Jest 29.x)
- `npm run lint` runs successfully (ESLint 8.x + TypeScript 5.x)
- `src/` and `tests/` directories exist at the project root

**Steps**:

1. Review the evolution history — list all contents of the `.claude/` directory to confirm that every artifact from the six-step process exists:
   ```bash
   find .claude -type f | sort
   ```
   You should see output similar to:
   ```
   .claude/agents/security-reviewer.md
   .claude/hooks/pre-commit-test.sh
   .claude/hooks/stop-validator.sh
   .claude/pitfalls.md
   .claude/settings.json
   CLAUDE.md
   ```
   If any artifact is missing, complete it before continuing. Skipping any step in the six-step process means there is a known risk point that hasn't been covered.

2. Cross-reference the six-step process to confirm each step has a corresponding artifact:
   - `CLAUDE.md` (50-100 lines) -> Step 1 complete
   - `.claude/settings.json` (contains deny rules) -> Step 2 complete
   - `.claude/hooks/stop-validator.sh` (Stop Validator) -> Step 3 complete
   - `.claude/agents/security-reviewer.md` (first Subagent) -> Step 4 complete
   - `.claude/pitfalls.md` (pitfall log with loop-closed entries) -> Step 5 complete
   Complete any missing steps before continuing.

3. Now add the final product feature. Use the following prompt in Claude Code:
   ```
   Create a generateTests function in src/test-gen.ts that meets these specifications:

   - Takes a TypeScript file path (string) as input
   - Uses the TypeScript compiler API or AST parsing to read all named export functions from the file
   - Generates a Jest test skeleton for each function, including:
     - A describe block (named after the module)
     - An it block for each function (described as "should [function name]")
     - A placeholder assertion of expect(true).toBe(true) inside each it block, with a TODO comment
   - Outputs the generated test content as a string, without writing directly to a file (the caller decides the output target)
   - Create corresponding tests in tests/test-gen.test.ts that verify at least two scenarios:
     1. Input containing a TypeScript file with a single export function; output should contain the corresponding describe and it blocks
     2. Input containing a file with no export functions; output should be an empty string
   ```

4. Observe whether the Stop Validator works as expected. When Claude finishes the implementation and is ready to conclude, the Stop Validator should automatically run `npm test`. Expected validation sequence:
   - Claude implements `src/test-gen.ts` and `tests/test-gen.test.ts`
   - Claude attempts to end the conversation
   - Stop Validator runs `npm test && npm run lint`
   - If tests pass, Stop Validator returns success, Claude ends normally
   - If tests fail, Stop Validator returns failure, Claude must fix and retry
   If the Stop Validator is not triggered, check whether the hook configuration in `.claude/settings.json` is correct.

5. Execute the Step 5 closed loop. If Claude made new mistakes during the implementation of `test-gen.ts`, record them in the pitfall log:
   ```bash
   # Open .claude/pitfalls.md and add a new entry at the end
   # Format: Date | Task | Incorrect Behavior | Severity | Loop Closure Method (or pending)
   ```
   Even if Claude performed perfectly, you should still record "Claude correctly implemented TypeScript AST parsing on the first attempt, no additional constraints needed" — this is positive data indicating that this type of task doesn't need Hook coverage in the future.

6. Execute the Step 6 audit. For each script in the `.claude/hooks/` directory and each Agent in `.claude/agents/`, answer the following questions one by one:

   **For each Hook**:
   - Has this Hook been triggered in the past 30 days? (If the Hook script doesn't write logs, add a line now: `echo "[$(date)] Hook triggered: $HOOK_NAME" >> .claude/hook-activity.log`)
   - Can Claude 4 or the latest version already avoid the problem this Hook intercepts on its own?

   **For each Agent**:
   - Has this Agent been delegated tasks in the last 30 days?
   - Does the Agent's prompt still accurately describe its scope of work? (As the main codebase evolves, the Agent's instructions may have become outdated)

   **For CLAUDE.md**:
   - Are there entries that were added three months ago but have never been validated as effective?
   - Are there entries whose corresponding problems are now covered by Hooks or ESLint rules, allowing them to be removed from CLAUDE.md (simplifying the index)?

   Record the audit results, remove confirmed ineffective components, and update the "last verified date" for components confirmed to still be effective.

**Expected Results**:
- `src/test-gen.ts` exists, contains the `generateTests` function, TypeScript compiles without errors
- `tests/test-gen.test.ts` exists, contains at least two test scenarios, `npm test` passes
- `.claude/pitfalls.md` has new entries (even if it's a "no new pitfalls" record)
- After the audit, the `.claude/` directory structure is streamlined: no Hooks that have never been triggered, no Agents with outdated descriptions
- For every file in the `.claude/` directory, you can clearly explain which pitfall or principle it corresponds to

**Common Mistakes**:

- **Pitfall log is not maintained, closed loop breaks**: Pitfalls are recorded when they happen, but no one goes back to distill them into rules, and the log becomes an unread stream of events. The solution is to set a fixed review reminder — for example, 15 minutes before leaving work every Friday, open `.claude/pitfalls.md` and process the "pending closure" entries. This is not a task that requires inspiration; it's a habit that requires discipline.

- **Can't bear to delete during audits, system only grows**: The psychological resistance to deleting Hooks or Agents comes from "What if I need it after deleting it?" This concern is usually unfounded: Git has version control, and deleted files can be recovered. A more useful question to ask is: "If this component disappears, the worst case is Claude makes an error that takes me how long to fix?" If the answer is "10 minutes," the component's cost of existence (maintenance, cognitive burden) is very likely higher than the protection it provides.

---

## Key Takeaways
- A Harness is not preventive design but pitfall-driven evolution — every rule should correspond to a real problem that has already occurred, not an imagined risk
- The order of the six-step process matters: bare-run observation comes first, constraint building comes after; the Stop Validator is added after basic constraints, and Subagents are split out only after context overload signals appear
- Among the three anti-patterns, "Constraint Accumulation Syndrome" is the hardest to detect because it accumulates slowly — each rule makes sense when added, but the problem only manifests after accumulation
- The pitfall closed loop is the mechanism that keeps the Harness effective: log -> analyze -> rule -> validate -> mark as closed. If any link breaks, the Harness stops evolving
- Periodic auditing (Step 6) is the only mechanism that keeps the Harness lean — model capabilities are improving, and without audits, the Harness will forever address problems from older model versions

## Self-Assessment
- Open `.claude/pitfalls.md` — when was the oldest "pending closure" entry added? How many days ago? If it's more than two weeks, the closed-loop mechanism is already broken.
- Is there any rule in your Harness where you can't explain which real pitfall it corresponds to? That rule is a symptom of Copy-Paste Syndrome and needs its necessity re-evaluated.
- If you upgrade Claude Code to the next major version tomorrow, how would you verify which Hooks can be removed? If you can't articulate this verification plan right now, Step 6 auditing hasn't truly been put into practice yet.

---

⬅️ [Previous: Seven Design Principles](06-1-seven-principles.md) ｜ 📖 [Index](../00-index.md) ｜ ➡️ [Next: Boris Cherny Workflow](../07-frameworks/07-1-boris-cherny.md)

🌐 [繁體中文版](../../zh/06-Harness-Engineering/06-2-從零建立Harness.md)
