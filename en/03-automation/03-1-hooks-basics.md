# 03-1 Hooks Basics
> Hooks are "guarantees," not "suggestions" — learn them and turn Claude Code's behavior into system-level enforcement.

## Learning Objectives
- Understand the fundamental difference in reliability between Hooks and CLAUDE.md rules
- Identify the 6 most practical Hook events and when they trigger
- Master the hooks configuration format and field meanings in `settings.json`
- Set up 3 concrete foundational Hooks for ai-dev-assistant and verify they work

---

## Content

### 1. Why You Need Hooks

You wrote in CLAUDE.md: "Run ESLint every time a TypeScript file is modified." The first week, Claude dutifully complied. By the third week, it started "forgetting" — especially in long conversations or sessions after compaction.

This is not a bug; it is by design. Rules in CLAUDE.md are inherently "suggestions" — the model incorporates them into its context and decides on its own whether to follow them. When the context window is saturated, or when the model judges that a rule conflicts with the current task, suggestions get skipped.

Hooks solve this reliability problem. Hooks are system-level interceptors — forcibly invoked by Claude Code's runtime before and after tool execution, regardless of the model's intent. The model cannot "decide not to run" a PostToolUse Hook, just as a program cannot decide to prevent the operating system's signal handler from firing.

**David Chu's Graduation Path**: When you notice Claude no longer consistently follows a certain rule in CLAUDE.md, that is the moment it should "graduate" into a Hook. Preferences and style stay in CLAUDE.md; safety, validation, and formatting get promoted to Hooks.

---

### 2. Deterministic vs. Advisory

| Attribute | CLAUDE.md Rule | Hook |
|-----------|---------------|------|
| Nature | Advisory | Mandatory |
| Execution | Model's discretion | System-level interception |
| Reliability | 80–95% | 100% |
| Scope | Current context | All sessions |
| Use case | Preferences, style, tone | Safety, validation, formatting |

The key row in this table is "Reliability." 80–95% does not sound bad, but in safety scenarios a 5% failure rate is unacceptable. If you have a rule that says "never modify `.env` files," you need 100%, not 95%.

---

### 3. Common Hook Events

Claude Code provides multiple Hook event points. The following focuses on the 6 most practical ones:

| Event | When It Triggers | Typical Use |
|-------|-----------------|-------------|
| `PreToolUse` | Before tool execution (can block the operation) | Protect sensitive files, validate command safety |
| `PostToolUse` | After tool execution (operation already completed) | Auto-format, write audit logs |
| `Stop` | When Claude completes a full response | Mandatory validation (Validator Pattern) |
| `SessionStart` | When a session starts or resumes from a compacted state | Load external context, re-inject core rules |
| `PermissionRequest` | Before a permission prompt appears | Auto-approve known safe operations |
| `UserPromptSubmit` | Before the user's prompt is submitted | Pre-process input, append fixed instructions |

`PreToolUse` is the most powerful event: it is the only Hook that can "prevent" an operation from happening. All other events trigger after the operation has already occurred and can only perform post-processing.

---

### 4. Hook Configuration Format

Hook settings live in the `hooks` field of `.claude/settings.json`. The structure is as follows:

```json
{
  "hooks": {
    "EventName": [
      {
        "matcher": "ToolName (optional)",
        "hooks": [
          {
            "type": "command",
            "command": "shell command to execute"
          }
        ]
      }
    ]
  }
}
```

The precise meaning of each field:

- **`EventName`** (outermost key): Corresponds to an event from the table above, such as `PreToolUse` or `PostToolUse`.
- **`matcher`**: A string to match against tool names. Supports `|` to separate multiple tools, e.g., `"Edit|Write"`. When omitted, it matches all tools.
- **`type`**: The handler type. This chapter only uses `"command"` (execute a shell command). Advanced types (such as `"mcp"`) are covered in 03-2.
- **`command`**: The actual shell command string to execute. The command receives JSON-formatted event data via stdin.

Note the nested structure: the outer array contains "rule groups," each with its own matcher; the `hooks` array is the list of handlers under that group. A single event can have multiple groups, and they execute from top to bottom.

---

### 5. Hook Input and Output

Hook commands receive JSON-formatted event data via stdin. Taking `PreToolUse` intercepting the `Edit` tool as an example:

```json
{
  "session_id": "abc123",
  "hook_event_name": "PreToolUse",
  "tool_name": "Edit",
  "tool_input": {
    "file_path": "src/index.ts",
    "old_string": "const x = 1",
    "new_string": "const x = 2"
  }
}
```

This means your Hook script can use `jq` or any JSON parsing tool to read `tool_input.file_path` and decide whether to block the operation.

**Exit codes are the only language Hooks use to communicate with the system:**

| Exit Code | Meaning | Additional Effect |
|-----------|---------|-------------------|
| `exit 0` | Allow the operation to continue | stdout content is returned to Claude as context |
| `exit 2` | Block the operation (only effective for PreToolUse) | stderr content is sent to Claude as feedback explaining why it was blocked |
| Other non-zero values | Treated as a Hook execution error | Default behavior depends on configuration (usually allows continuation) |

The stderr message from `exit 2` is critically important — Claude reads it and adjusts its strategy accordingly. Write clear error messages so Claude knows "why it was blocked," not just "that it was blocked."

---

## Hands-On Example: Setting Up 3 Foundational Hooks for AI Dev Assistant

> Continuing the main case study: In the AI Dev Assistant project, you have already completed the basic settings.json configuration (02-2). Now you will promote three key rules from "suggestions" to "guarantees."

**Goal**:
Add 3 Hooks to ai-dev-assistant:
1. `PostToolUse`: Automatically run ESLint fix after editing TypeScript files
2. `PreToolUse`: Protect `.env` family files from modification
3. `SessionStart`: Re-inject core technical rules after compaction

**Prerequisites**:
- Completed 02-2 (`.claude/settings.json` already created)
- Project root already has `package.json` with ESLint 8.x installed
- `jq` is installed on the system (used for parsing JSON stdin)

**Steps**:

**Step 1**: Create the hooks directory.

```bash
mkdir -p .claude/hooks
```

**Step 2**: Create the Hook script to protect sensitive files at `.claude/hooks/protect-files.sh`.

```bash
#!/bin/bash
# Read event JSON from stdin
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

# Define protected file patterns
PROTECTED=(".env" ".env.local" ".env.production" ".env.staging")

for pattern in "${PROTECTED[@]}"; do
  if [[ "$FILE_PATH" == *"$pattern"* ]]; then
    echo "Blocked: cannot modify '$FILE_PATH' — this is a protected environment file." >&2
    echo "Suggestion: if you need to update environment variables, describe the change and let the developer apply it manually." >&2
    exit 2
  fi
done

exit 0
```

**Step 3**: Set the script's execute permission (missing this step causes the Hook to fail silently).

```bash
chmod +x .claude/hooks/protect-files.sh
```

**Step 4**: Open `.claude/settings.json` and add the `hooks` field after the existing `permissions` configuration. The complete structure is as follows:

```json
{
  "permissions": {
    "allow": ["Read", "Edit", "Write", "Bash"],
    "deny": []
  },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "FILE=$(echo \"$(cat)\" | jq -r '.tool_input.file_path // empty'); if [[ \"$FILE\" == *.ts || \"$FILE\" == *.tsx ]]; then npx eslint --fix \"$FILE\" 2>/dev/null; fi; exit 0"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/protect-files.sh"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "matcher": "compact",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Context restored after compaction. Project: ai-dev-assistant. Stack: Node.js 18.x, TypeScript 5.x, Jest 29.x, ESLint 8.x. Critical rules: (1) run npm test before marking any task complete, (2) never modify .env files directly, (3) all new files must have corresponding .test.ts files.'"
          }
        ]
      }
    ]
  }
}
```

**Step 5**: Open the ai-dev-assistant project in Claude Code and ask Claude to edit any `.ts` file (for example, add a line of code in `src/index.ts`).

Observe the terminal output: ESLint should automatically run after the edit and fix any formatting issues.

**Step 6**: Ask Claude to modify a `.env` file (for example, "Please add `LOG_LEVEL=debug` to .env").

Observe Claude's response: it should receive the "Blocked" message and change its strategy (such as telling you what you should add manually instead).

**Step 7**: Run the `/hooks` command to confirm all Hooks are loaded correctly and verify the events and matchers display correctly.

**Expected Results**:
- After editing any `.ts` or `.tsx` file, ESLint `--fix` runs automatically and silently corrects formatting issues
- When attempting to modify `.env`, `.env.local`, or similar files, Claude receives a clear blocking message and provides an alternative suggestion
- `/hooks` output shows 3 Hooks enabled, mounted on the `PostToolUse`, `PreToolUse`, and `SessionStart` events respectively
- After compaction and session restart, Claude can read the tech stack reminder from the `SessionStart` Hook's stdout

**Common Errors**:

**Error 1: `jq: command not found`**

Symptom: The Hook script fails immediately on execution, and Claude receives no valid blocking message or allow signal.

Cause: `jq` is not installed. Most Linux/macOS environments do not include jq by default.

Solution:
- macOS: `brew install jq`
- Ubuntu/Debian: `sudo apt install jq`
- Windows (Git Bash): Download the `.exe` from [https://jqlang.github.io/jq/](https://jqlang.github.io/jq/) and add it to PATH

Verification: `jq --version` should output a version number.

**Error 2: Hook script lacks execute permission**

Symptom: The `PreToolUse` Hook fails silently — `.env` is not protected, the operation proceeds normally, and there are no error messages.

Cause: `protect-files.sh` is missing the executable bit, so the shell cannot execute it. In some versions, the Hook system silently skips scripts that cannot be executed rather than reporting an error.

Solution: Confirm you have run `chmod +x .claude/hooks/protect-files.sh`. Use `ls -la .claude/hooks/` to verify the permission column shows `-rwxr-xr-x`.

**Error 3: ESLint's non-zero exit code causes the operation to be judged as "blocked"**

Symptom: After Claude edits a `.ts` file, the system reports the operation was blocked, even though ESLint only found lint issues rather than actual errors.

Cause: ESLint returns exit code 1 when it detects lint violations, not 0. If the PostToolUse Hook's `command` ends with ESLint's exit code, the system may incorrectly interpret it as a Hook execution error.

Solution: The `command` string must end with `; exit 0` to force the Hook to finish with a success status. This way ESLint's fixes are applied without affecting the allow/block judgment of the operation.

---

### Advanced Tip: Turning Linter Error Messages into Repair Instructions

A design detail overlooked by most people was revealed by OpenAI's Harness Engineering practice: custom linter error messages can be specifically designed for agents.

Traditional linter error:
```
error: Function exceeds maximum line count (50)
```

Linter error designed for agents:
```
error: Function exceeds 50 lines. Split into smaller functions, each handling
a single responsibility. Extract helper functions to the same module.
Run `npm run lint` to verify after splitting.
```

Because a Hook's stderr output is returned directly to Claude as context, well-crafted error messages effectively auto-inject repair instructions every time a violation occurs. This turns PostToolUse Hooks from merely "intercepting" into "intercepting + teaching."

In ESLint, you can achieve this effect by customizing a rule's `message` field. The investment is small (changing a few lines of error messages), but the impact on the agent's repair success rate is significant.

---

## Key Takeaways

- CLAUDE.md rules have 80–95% reliability; Hooks enforce at 100% — these are not competing approaches but tools at different levels
- `PreToolUse` is the only event that can block an operation; all other events can only perform post-processing
- Exit codes are the only language Hooks use to communicate with the system: `exit 0` allows, `exit 2` blocks, and stderr messages tell Claude the reason
- `matcher` gives you precise control over a Hook's trigger scope, avoiding unnecessary delays across all tools
- Non-zero exit codes from tools like ESLint must be explicitly overridden with `; exit 0`, or they will cause unexpected blocking behavior

---

## Self-Check

- If you want to ensure "every time Claude creates a new file, it is automatically staged in git," which Hook event should you use? What should the matcher be set to?
- When a `PreToolUse` Hook script executes `exit 2`, how does Claude know "why" it was blocked? What happens if you do not write any stderr message?
- What are the most suitable scenarios for CLAUDE.md rules vs. Hooks respectively? Give an example from ai-dev-assistant of one rule that should stay in CLAUDE.md and one that should be promoted to a Hook, and explain the reasoning.

---

⬅️ [Previous: Agent = Model + Harness](../02-config/02-3-harness-preview.md) ｜ 📖 [Index](../00-index.md) ｜ ➡️ [Next: Advanced Hooks & Validator Pattern](03-2-hooks-advanced.md)

🌐 [繁體中文版](../../zh/03-自動化層/03-1-Hooks基礎.md)
