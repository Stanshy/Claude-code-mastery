# 04-2 Context Management Strategies
> Proactively manage Claude Code's context window so that every token is spent on information that truly matters.

## Learning Objectives
- Understand the relationship between context window fill rate and performance degradation
- Identify the 4 early symptoms of context overload and intervene before quality drops
- Master 8 specific management techniques, especially the Document & Clear method
- Design workflows that avoid blowing the context in complex, multi-stage tasks

---

## Main Content

### 1. Why Context Management Is the Most Important Skill

Claude Code has a 200K token context window. That number looks generous, but in real development work the fill rate is far faster than intuition suggests.

A medium-sized task — for example, adding a feature to `src/reviewer.ts` that involves reading 10 related files, 5 tool call outputs, and 3 rounds of revision based on feedback — can easily consume 50,000 to 80,000 tokens. If the task also includes investigating a bug plus test execution output, exceeding 100K is not uncommon.

**The relationship between performance and usage is not linear.** After context usage exceeds 40%, response quality begins to noticeably decline; after 80%, Claude's ability to handle complex reasoning is significantly limited; at 95%, the system triggers automatic compaction, but compaction is lossy — it preserves structure while discarding details.

Core principle: Automatic compaction is a safety net, not a strategy. By the time compaction is passively triggered, you have already let Claude work in a degraded state for a while. Proactively managing context is superior to passively waiting for compaction in both work quality and token efficiency.

---

### 2. The 4 Early Symptoms of Context Overload

Identifying overload is more important than waiting for the system to trigger. The following 4 symptoms are observable warning signs:

**Symptom 1: Instruction Forgetting**

You told Claude at the beginning of the session that "all functions must have JSDoc comments," but after 10 rounds of conversation, Claude starts producing functions without JSDoc. This is not Claude's fault — the early instructions in the context have been diluted by subsequent tool call outputs.

**Symptom 2: Previously Fixed Errors Reappearing**

You corrected a naming convention issue in round 5, but in round 15 Claude reverts to the original incorrect naming. This is a sign that the critical correction record in the context has been pushed out of the effective range.

**Symptom 3: Responses Becoming Shorter with Fewer Details**

Claude's responses shift from detailed implementation explanations to increasingly brief summaries. This is the result of Claude making trade-offs under limited context — it prioritizes preserving core reasoning at the expense of detailed output.

**Symptom 4: Declining Success Rate for the Same Prompt**

An instruction that executed correctly every time early in the session becomes unstable later, sometimes succeeding and sometimes failing. This is inconsistent behavior caused by incomplete background information in the context.

When any of these symptoms appears, you should immediately take management measures rather than continuing to retry the same prompt.

---

### 3. 8 Management Techniques

| Technique | What to Do | When to Use | Savings |
|---|---|---|---|
| `/clear` | Completely reset context, start a fresh session from zero | When switching to an unrelated task, or after completing a full stage | 100% (full reset) |
| `/compact [instructions]` | Compress context history, retaining a summary | Approaching the window limit but the task is not finished and you cannot `/clear` | 50-70% savings after compaction |
| Subagent Isolation | Delegate complex investigations to a Subagent | Any research or debugging task requiring many intermediate steps | Main conversation does not bloat at all |
| `/btw` | Side-question mode; the answer does not enter conversation history | Need to quickly clarify a small question without adding to context | No context increase |
| Plan Mode | Explore and analyze in read-only mode without executing any modifications | Requirements understanding, solution evaluation, and architecture planning stages | 40-60% token savings |
| Document & Clear | Write the plan to a .md file → `/clear` → reference the file to continue | Standard workflow for complex multi-stage feature development | Each stage starts with a clean context |
| Short Sessions | Proactively end the session after 30-45 minutes and restart | Time rhythm management for daily development | Prevents silent context accumulation |
| CLAUDE.md Streamlining | Keep CLAUDE.md under 200 lines with only essential instructions | Always applicable; review periodically after initial creation | Reduces per-session load consumption |

---

### 4. The Document & Clear Method: The Single Most Important Technique

Document & Clear is the most common workflow used by high-output engineers when handling complex features with Claude Code. This method was originally popularized by Boris Cherny within Anthropic and later adopted and validated by multiple engineers.

**Four Steps**:

**Step A: Have Claude Complete a Stage of Work**

Define a clear boundary for a "stage." For example: "Analyze the existing architecture and plan the implementation approach for the new feature" is one stage; "Implement the feature based on the plan" is the next stage. Do not let a single session bear the entire end-to-end workload.

**Step B: Have Claude Write the Plan, Progress, and Key Decisions into a .md File**

Before the session ends, ask Claude to organize all important information produced during this stage into a document. This document should include: what was decided (with reasoning), what was discovered (with key data), and what to do next (with specific instructions). The document must be self-contained — a Claude with no memory of this session should be able to seamlessly continue work after reading this document.

**Step C: `/clear` to Reset the Context**

Cleanly end the session. All intermediate artifacts, investigation processes, and trial-and-error records are cleared together.

**Step D: In a Fresh, Clean Context, Reference the File with `@filename.md` to Continue Work**

At the start of the new session, directly reference the document: "Based on the plan in @docs/plan-analyzer.md, continue with phase two implementation." Claude executes the subsequent work at high quality within the full 200K context space.

**Why This Is Better Than Letting Context Accumulate**: Each stage executes in a clean context, keeping Claude's reasoning quality at a high level; the existence of the document makes the entire development process trackable and reproducible; if you ever need to go back and revise, the document records the reasoning behind every decision.

---

### 5. Automatic Compaction: Behavior and Boundaries

Understanding the mechanics of automatic compaction helps you see why you should not rely on it.

**Trigger Condition**: Context usage reaches 95% of the 200K window.

**Compaction Behavior**: The system retains the contents of `CLAUDE.md`, key messages from the most recent conversation rounds, and the summary structure of the conversation. Everything else — detailed tool call outputs, intermediate reasoning steps, and early specific instructions — is compressed or discarded.

**The Cost of Compaction**: After compaction, Claude's grasp of earlier decisions, instructions, and context becomes fuzzy. If you established an important understanding early in the session (e.g., "the edge case for this function is X"), Claude may no longer remember it after compaction.

**The Right Framing**: Automatic compaction is a safety net that prevents the session from completely collapsing. It lets you continue the conversation before context is fully exhausted. But it is not a reason to keep working at 80% context usage — that usage level has already exceeded the tipping point for quality decline.

---

## Practical Example: Context Management for a Major Feature in the AI Dev Assistant
> Main case study continued: In the AI Dev Assistant project, use the Document & Clear method to develop the analyzeFile feature while keeping context usage low throughout.

**Goal**: Add an `analyzeFile` function to `src/reviewer.ts`, demonstrating how to complete a full feature — requiring code analysis, planning, implementation, and testing — without blowing the context.

**Prerequisites**: Chapter 04-1 (Custom Subagents) is completed, and the ai-dev-assistant project has `src/reviewer.ts` and its corresponding test file.

**Steps**:

**Step 1**: Launch Claude Code and Switch to Plan Mode

```bash
claude
```

After entering interactive mode, press `Shift+Tab` twice to switch to Plan Mode (the bottom status bar displays Plan Mode).

**Step 2**: Perform Requirements Analysis in Plan Mode

Enter:

```
Analyze the existing structure of AI Dev Assistant, understand the architecture of
src/reviewer.ts, and plan how to add an analyzeFile(filePath: string): Promise<ReviewResult>
function. Include: function signature, integration points with existing functions, type
definitions to add, and a test case list.
```

The benefit of Plan Mode: Claude reads and analyzes all relevant files without executing any modification operations. This stage only consumes the tokens needed for analysis, and tool call outputs do not fill up the context.

**Step 3**: Have Claude Write the Plan into a Document

After Plan Mode analysis is complete, switch back to Normal Mode (press `Shift+Tab` once), then enter:

```
Write this analysis result and implementation plan to docs/plan-analyzeFile.md.
The document must include: existing architecture summary, complete function signature for
analyzeFile, integration point descriptions, types to add, test case list (at least 5),
and edge cases to watch out for during implementation.
```

Wait for Claude to create `docs/plan-analyzeFile.md`.

**Step 4**: Reset the Context

```
/clear
```

All intermediate artifacts, analysis processes, and Plan Mode read records generated during this session are completely cleared. At this point, `docs/plan-analyzeFile.md` is already saved on disk and is unaffected by `/clear`.

**Step 5**: Begin Implementation in a Clean Context

```
Based on the plan in @docs/plan-analyzeFile.md, implement the analyzeFile function
in src/reviewer.ts. Implement the function body first, then add corresponding Jest tests.
```

Claude reads `docs/plan-analyzeFile.md` and executes the implementation at high quality within the full 200K context space.

**Step 6**: Run Tests to Confirm Results

After Claude completes the implementation, confirm that tests pass:

```
Run npm test -- --testPathPattern="reviewer" and report the results
```

**Step 7**: If Tests Fail, Delegate Investigation to a Subagent

If any tests fail, do not expand the investigation in the main conversation (this would consume context). Instead, use:

```
@debugger Investigate the failing tests related to analyzeFile in src/reviewer.test.ts, fix them, and confirm all tests pass
```

The debugger Agent completes the investigation and fix in an isolated context, and the main conversation only receives a summary of "root cause, fix applied, tests passing."

**Step 8**: Record Completion Status (Preparing for the Next Stage)

After the feature is complete, have Claude update the document:

```
Update docs/plan-analyzeFile.md by adding an "Implementation Complete" section at the end,
recording: the final function signature (if adjusted), test pass rate, and any differences
discovered from the original plan.
```

**Expected Results**:
- The `analyzeFile` function is fully implemented and all Jest tests pass
- The main conversation's context usage stays low throughout the entire process: Plan Mode analysis, `/clear` reset, and Subagent isolation work together
- `docs/plan-analyzeFile.md` remains in the project as a complete development record, available for future reference or as a basis for PR descriptions

**Common Mistakes**:

**Mistake 1: Forgetting to reference the plan document after `/clear`, re-explaining requirements from scratch**

This is the most common error. Saying "continue implementing analyzeFile" directly after `/clear` means Claude does not know what analyzeFile is and needs all background re-explained, wasting tokens and easily missing details.

Correct approach: Build the habit of confirming the document exists before `/clear` and referencing it with `@filename.md` as the first message after `/clear`. You can confirm with one sentence: "Before /clear, has the document been written to docs/plan-analyzeFile.md?"

**Mistake 2: Attempting to have Claude edit files while in Plan Mode, and the operation is rejected**

Plan Mode is read-only. Claude can only read and analyze; any modification operations (Edit, Write, executing Bash commands) will be rejected.

Correct approach: Use Plan Mode only for analysis and planning. After confirming the plan, switch back to Normal Mode (`Shift+Tab` once) before asking Claude to write the plan to a file, then proceed with implementation. Remember the switching order: Plan Mode for analysis → Normal Mode for writing the document → `/clear` → Normal Mode for implementation.

---

## Key Takeaways
- Context performance degradation is non-linear: decline begins after 40%, significant limitation after 80% — do not wait until 95% to act
- The 4 early symptoms (instruction forgetting, error reappearing, shorter responses, declining success rate) are warning signs that appear earlier than system prompts
- Document & Clear is the most efficient strategy: let every development stage execute in a clean context, with documents bridging continuity across sessions
- Automatic compaction is a safety net, not a working strategy: relying on it means you have already let Claude work in a degraded state
- Subagent isolation and Plan Mode are the two most powerful tools for proactive context management, and the two can be combined

## Self-Check
- You observe that Claude starts producing TypeScript functions without type annotations during a session, even though you explicitly required "all functions must have complete type annotations" at the beginning of the session. What symptom is this? What is your next action?
- In the Document & Clear method, regarding the requirement that "the document must be self-contained," how would you design the structure of `docs/plan-analyzeFile.md` to ensure that a Claude with no memory of the current session can seamlessly continue work after reading it?
