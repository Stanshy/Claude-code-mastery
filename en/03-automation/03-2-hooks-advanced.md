# 03-2 Advanced Hooks & Validator Pattern
> If you only learn one advanced capability, let it be this chapter. The Stop Hook Validator Pattern is the key mechanism that transforms Claude Code from "might make mistakes" to "guaranteed correct."

## Learning Objectives
- Understand the mathematical principles behind the Stop Hook Validator Pattern, and why it is the most important design pattern
- Master the purpose of the `stop_hook_active` field to avoid infinite loops
- Learn about the four Hook handler types (command, prompt, agent, http) and their applicable scenarios
- Learn to combine multiple Hooks to form a chained validation pipeline

---

## Main Content

### 1. Why the Validator Is the Most Important Design Pattern

David Chu's core insight is worth quoting directly: **A good validator with a bad workflow beats a good workflow with no validator.**

This statement seems counterintuitive, but the math fully supports it.

Suppose you have a 5-step automated workflow where each step has an 80% success rate. The overall success rate is:

```
0.8 × 0.8 × 0.8 × 0.8 × 0.8 = 0.8^5 ≈ 33%
```

You spend a great deal of time polishing the workflow, yet two-thirds of the execution results are failures.

Now consider an alternative approach: add a Stop Hook Validator. Even if each individual step only has 70% reliability, but you allow 4 retries, the success rate becomes:

```
1 - 0.3^4 = 1 - 0.0081 ≈ 99%
```

The mathematical essence of a validator is: **it transforms "serial failure" into "parallel retries."** In a serial system, failure at any single point means total failure; in a parallel retry system, as long as one attempt succeeds, it counts as a pass. This structural difference is why validators take priority over workflow optimization.

---

### 2. Stop Hook Validator Pattern

The Stop Hook is the most strategically valuable of all Hook types. It intercepts the moment when "Claude believes the task is complete and is about to stop."

**How It Works (Complete Flow):**

1. Claude finishes executing the task and is about to stop
2. The Stop Hook intercepts and runs validation (e.g., `npm test` + `npm run lint`)
3. If validation fails -> the failure message is sent back to Claude -> Claude automatically fixes the issue -> attempts to stop again
4. If validation passes -> stopping is allowed, and the task ends

**Key Mechanism: The `stop_hook_active` Field**

This is the most easily overlooked and most dangerous detail. When a Stop Hook fires and Claude fixes the problem and attempts to stop again, Claude Code adds `stop_hook_active: true` to the JSON input passed to the Hook. Your Hook script **must** `exit 0` immediately upon detecting this field; otherwise the Hook will trigger validation again, creating an infinite loop.

The complete loop-prevention Stop Validator script is as follows:

```bash
#!/bin/bash
INPUT=$(cat)

# Prevent infinite loop: if already inside a stop hook, allow through immediately
STOP_ACTIVE=$(echo "$INPUT" | jq -r '.stop_hook_active // false')
if [ "$STOP_ACTIVE" = "true" ]; then
  exit 0
fi

# Run validation
cd "$CLAUDE_PROJECT_DIR"
npm test 2>&1
TEST_RESULT=$?

npm run lint 2>&1
LINT_RESULT=$?

if [ $TEST_RESULT -ne 0 ] || [ $LINT_RESULT -ne 0 ]; then
  echo "Validation failed. Fix the issues before completing." >&2
  exit 2
fi

exit 0
```

There are three design decisions in this script worth explaining:

- `cd "$CLAUDE_PROJECT_DIR"`: Ensures npm commands execute in the correct project root directory; otherwise `node_modules` cannot be found
- `echo ... >&2`: Error messages must be written to stderr so that Claude Code can read them and pass them back to Claude; stdout is reserved for JSON communication
- `exit 2`: Exit code 2 means "block the operation and return the error message"; exit 1 means "block but do not return a message"; exit 0 means "allow to continue"

---

### 3. Four Hook Handler Types

Chapter 03-1 only covered the `command` type. There are four complete Hook handler types, each addressing different levels of validation needs:

| Type | Description | Use Cases |
|------|-------------|-----------|
| `command` | Execute a shell command | Formatting, file protection, validation scripts |
| `prompt` | Use the Haiku model for single-turn semantic judgment | Semantic checks (e.g., "Does this change comply with the spec?") |
| `agent` | Use a sub-agent for multi-turn validation | Complex validation (running tests, checking coverage, code review) |
| `http` | POST JSON to an HTTP endpoint | External system integration (audit logs, Slack notifications) |

**`prompt` Type Example**

Suitable for scenarios that require semantic understanding but do not need to execute programs. For example, confirming that every modified file has a corresponding test file:

```json
{
  "type": "prompt",
  "prompt": "Check if all modified files have corresponding test files. Return {\"ok\": true} or {\"ok\": false, \"reason\": \"...\"}"
}
```

This type uses the Claude Haiku model, which is fast and low-cost, making it suitable for high-frequency events (such as PostToolUse).

**`agent` Type Example**

Suitable for complex validations that require multi-step reasoning or executing multiple tools. For example, running a full test suite and analyzing the results:

```json
{
  "type": "agent",
  "prompt": "Run the full test suite and verify all tests pass. If any fail, report which tests failed and why.",
  "timeout": 120
}
```

The `timeout` field (in seconds) is essential because test suites can take a long time to run. An agent hook without a timeout setting can cause the entire flow to hang.

**`http` Type Example**

Suitable for scenarios requiring external system integration. Each time a Hook is triggered, Claude Code sends the event data as a POST JSON request:

```json
{
  "type": "http",
  "url": "http://localhost:8080/hooks/audit",
  "headers": {
    "Authorization": "Bearer $TOKEN"
  },
  "allowedEnvVars": ["TOKEN"]
}
```

`allowedEnvVars` is a security mechanism: only environment variables on this allowlist will be expanded. Hardcoding secrets directly in the JSON is incorrect practice.

---

### 4. Hook Composition Design Patterns

Multiple Hook handlers can be configured for the same Hook event, forming a chained pipeline. Each handler executes in order, and any single `exit 2` will block the operation and return an error.

**PostToolUse Chaining Example:**
- First Hook: Formatting (Prettier)
- Second Hook: Lint check (ESLint)
- Third Hook: Log the operation (http type)

**Stop Chaining Example:**
- First Hook: Run tests (`npm test`)
- Second Hook: Type checking (`npm run type-check`)
- Third Hook: Verify test coverage meets the threshold

Design principle: **Place faster checks earlier.** Formatting and linting only take a few seconds; placing them first allows quick filtering of obvious issues before time-consuming tests, saving overall execution time.

---

### 5. Hook Troubleshooting

Hook issues are usually not easy to spot directly, because many failures are silent. Here are the five most common problems and their solutions:

| Problem | Cause | Solution |
|---------|-------|----------|
| Hook not triggering | Typo in matcher | Use the `/hooks` command to verify configuration; note that matchers are case-sensitive |
| JSON parse failure | Extra `echo` output in `.bashrc` or `.zshrc` | Ensure the shell profile does not output any non-JSON content in non-interactive mode |
| Hook timeout | Script execution exceeds the default limit | Add a `timeout` field (in seconds) to the configuration; optimize script performance |
| Infinite loop | Stop Hook does not check `stop_hook_active` | Must execute `exit 0` when `stop_hook_active = true` |
| Hook fails silently | Script lacks execute permissions | Run `chmod +x` on all hook scripts |

Among these, "JSON parse failure" is the most insidious problem. Claude Code communicates with Hook scripts via stdout using JSON. If the shell profile (`.bashrc`, `.zshrc`) outputs any text on startup (e.g., `echo "Welcome"` or initialization messages from various tools), that text gets mixed into stdout, causing JSON parsing to fail -- yet you will see almost no error indication.

---

## Hands-On Example: Adding a Stop Validator to the AI Dev Assistant

> Continuing the main case study: In the AI Dev Assistant project, we have already completed the basic Hook setup (formatting and file protection). Now we will add a Stop Hook to ensure that Claude can never finish a task while tests are failing.

**Goal**: Add a Stop Hook so that Claude must pass `npm test` and `npm run lint` before completing any task

**Prerequisites**:
- Chapter 03-1 is complete (`.claude/settings.json` already exists with basic Hooks configured)
- The project root directory is `/projects/ai-dev-assistant`
- `package.json` already has `test` and `lint` scripts

**Steps:**

1. Create the validation script `stop-validator.sh` in the `.claude/hooks/` directory, using the complete script provided in Section 2 of this chapter (including the `stop_hook_active` guard logic).

2. Grant the script execute permissions:
   ```bash
   chmod +x /projects/ai-dev-assistant/.claude/hooks/stop-validator.sh
   ```

3. Open `.claude/settings.json` and add the `Stop` event to the `hooks` object:
   ```json
   "Stop": [
     {
       "hooks": [
         {
           "type": "command",
           "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/stop-validator.sh"
         }
       ]
     }
   ]
   ```
   Note that the `command` path uses the `$CLAUDE_PROJECT_DIR` environment variable to ensure the script can be correctly located from any working directory.

4. Save `settings.json`, then launch Claude Code:
   ```bash
   cd /projects/ai-dev-assistant && claude
   ```

5. Give Claude a task that will cause tests to fail:
   ```
   Add a bug to the main function in src/index.ts: change the return value to undefined
   ```

6. Observe the execution flow: Claude completes the modification and attempts to stop -> Stop Hook triggers -> `npm test` fails -> Claude receives the stderr message -> Claude automatically fixes the bug -> attempts to stop again -> Stop Hook triggers again -> `stop_hook_active` is true -> direct `exit 0` -> task ends normally.

7. Confirm the Stop Hook trigger records in the terminal output, and run `npm test` to verify all tests pass.

**Expected Results**:
- Claude cannot finish a task while tests are failing, even if Claude "believes" it is done
- Claude will automatically detect validation failures and perform fixes without human intervention
- The terminal will show two (or more) Stop Hook trigger records
- The final code state will have all tests passing and zero lint warnings

**Common Mistakes:**

- **Infinite loop (Claude keeps trying to stop but is continuously blocked)**: The cause is almost certainly a missing `stop_hook_active` check in the script. Verify that the first logic block in the script correctly reads and evaluates this field. Use `echo "$INPUT" | jq '.'` in the script to debug and confirm the actual structure of the JSON input.

- **`npm test` cannot find `node_modules` inside the Hook**: The Hook script's working directory is not necessarily the project root. The script must include `cd "$CLAUDE_PROJECT_DIR"`. Verify that the `$CLAUDE_PROJECT_DIR` environment variable has the correct value (you can add `echo "$CLAUDE_PROJECT_DIR" >&2` to the script for debugging).

- **Hook executes but Claude does not receive the error message**: Confirm that error messages are written to stderr (`echo "..." >&2`), not stdout. stdout is the channel Claude Code uses for JSON communication; any non-JSON stdout output may cause parsing issues and will not be treated as an error message.

---

## Key Takeaways
- The core value of a Validator is transforming serial failure into parallel retries; even with lower per-step reliability, overall reliability can approach 100%
- `stop_hook_active: true` is the only mechanism to prevent Stop Hook infinite loops; all Stop Validator scripts must handle this field
- The four Hook types (command, prompt, agent, http) correspond to four different levels of validation complexity; choose based on the principle of "if command can solve it, don't use agent" -- the minimum complexity principle
- When chaining multiple Hooks, the performance ordering principle is: fast checks (formatting, lint) go first, time-consuming checks (tests, coverage) go last
- Error messages must be written to stderr, with stdout reserved for JSON communication; violating this rule causes silent failures that are extremely difficult to debug

## Self-Check
- If your Stop Hook script lacks the `stop_hook_active` check, what would happen during actual execution? Can you describe the complete loop process?
- In a Hook configuration with a PostToolUse event, how would you decide between using the `command` type versus the `prompt` type? What are the criteria for each?
- Building on the `ai-dev-assistant` configuration from the previous chapter, your `settings.json` should now have Hooks for both `PostToolUse` and `Stop` events. If both events include lint checking, how would you refactor to avoid duplication?

---

⬅️ [Previous: Hooks Basics](03-1-hooks-basics.md) ｜ 📖 [Index](../00-index.md) ｜ ➡️ [Next: Skills System](03-3-skills.md)

🌐 [繁體中文版](../../zh/03-自動化層/03-2-Hooks進階與Validator模式.md)
