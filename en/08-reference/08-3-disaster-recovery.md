# 08-3 Disaster Recovery & Debug Guide
> A systematic troubleshooting process for when Claude Code breaks — from hangs to crashes to context explosions.

## Learning Objectives
- Classify Claude Code failures into 5 fault types and identify each by its observable symptoms
- Execute a step-by-step recovery process for hung sessions, context explosions, and Hook errors
- Diagnose MCP server connection failures using manual curl tests and config inspection
- Use `--resume`, `Esc+Esc`, and `claude -c` to recover work from interrupted sessions

---

## Content

### 1. Fault Classification & Diagnostic Mindset

Before debugging, identify what category the problem falls into. Each fault type has a different first response — applying the wrong fix wastes time and can make things worse.

| Fault Type | Observable Symptoms | Severity | First Action |
|---|---|---|---|
| **Hang / Unresponsive** | Spinner frozen, no output for 60+ seconds, terminal unresponsive to Enter | Medium | `Ctrl+C`, then check if the process is still running |
| **Context Explosion** | Instructions forgotten, previously fixed bugs reappear, responses become terse | High | Run `/cost` to check token usage, then `/compact` or `/clear` |
| **Hook Error** | Operations not blocked when they should be, unexpected "Hook error" messages, silent failures | Medium | Run `/hooks` to list active hooks, then test the script manually |
| **MCP Connection Failure** | `/mcp` shows server as disconnected, tool calls return connection errors | Medium | Run `/mcp` to check status, then test the server endpoint manually |
| **Worktree Conflict** | `fatal: '<path>' is already checked out`, branch lock errors, merge conflicts across worktrees | Low | Run `git worktree list` to see all worktrees and their states |

**Diagnostic mindset**: Always gather information before acting. Run the diagnostic command first (`/cost`, `/hooks`, `/mcp`, `git worktree list`), read the output, then decide on the fix. Do not restart the session as a first resort — you lose context and undo history.

---

### 2. Claude Code Hang / Unresponsive

Two situations look similar but require different responses:

**Spinner stopped, no output, terminal does not respond to Enter**

This is a genuine hang. The process is stuck — waiting for a network response that will not arrive, or blocked on a system call.

Recovery steps:

```bash
# Step 1: Press Ctrl+C in the Claude Code terminal.
# This sends SIGINT and usually recovers the session gracefully.
# Claude will print "Interrupted" and return to the prompt.

# Step 2: If Ctrl+C does not work within 5 seconds, open a second terminal:
ps aux | grep claude

# Step 3: Kill the stuck process:
kill -9 <PID>

# Step 4: Resume the session:
claude -c
```

After killing the process, `claude -c` resumes the most recent conversation. You lose only the in-flight operation, not the conversation history.

**Spinner still spinning, but response is taking 2+ minutes**

This is not a hang. Claude is processing a task that involves extended thinking or multiple tool calls. Check the terminal status line — if it shows "Thinking..." or a tool name, the process is active.

**When to wait vs. when to kill:**

| Indicator | Action |
|---|---|
| Status line shows "Thinking..." and updates periodically | Wait — extended thinking on complex tasks can take 2-3 minutes |
| Status line shows a Bash command running (e.g., `npm test`) | Wait — the command is executing; check if the underlying process is active |
| No status line update for 60+ seconds, no network activity | Kill and resume with `claude -c` |
| Terminal completely frozen (cannot type, no cursor movement) | Kill the terminal process, reopen terminal, run `claude -c` |

---

### 3. Context Explosion Recovery

Context explosion is the most common failure mode in long sessions. For the 4 symptoms that signal context overload (instruction forgetting, previously fixed errors reappearing, responses becoming shorter, declining success rate for the same prompt), see the detailed descriptions in [04-2 Context Management](../04-orchestration/04-2-context-management.md).

This section covers recovery — what to do after context overload has already degraded your session.

**Decision tree: `/compact` rescue vs. `/clear` restart**

```
Context overload detected
    │
    ├── Is the current task still in progress?
    │     ├── YES → Is Claude still producing correct output (just slower)?
    │     │     ├── YES → /compact with instructions to preserve current task state
    │     │     └── NO  → Document progress to a file, then /clear and restart
    │     └── NO  → /clear (task is done, start fresh)
    │
    └── Did automatic compaction already trigger?
          ├── YES → Check if critical context survived. If not, /clear and reload from files.
          └── NO  → You still have time. Run /compact proactively.
```

**`/compact` rescue — preserving in-flight work:**

```
/compact Preserve: (1) current task is implementing auth middleware in src/middleware/auth.ts, (2) the token validation logic is complete but error handling is not, (3) test file is at src/middleware/__tests__/auth.test.ts with 3 passing tests
```

The instructions you pass to `/compact` tell the compaction algorithm what to prioritize. Without instructions, compaction preserves structure but may discard the specific details you need most.

**Context budget allocation**

Context budget allocation is a planning technique — not a Claude Code feature, but a discipline you apply before starting work. The idea: divide your 200K context window into budgets per task phase, and enforce `/compact` or `/clear` boundaries between phases.

| Phase | Budget | Action at Boundary |
|---|---|---|
| Planning & investigation | 40K tokens (~20% of window) | Write plan to `docs/plan.md`, then `/clear` |
| Implementation | 80K tokens (~40% of window) | `/compact` with implementation state summary |
| Testing & debugging | 50K tokens (~25% of window) | `/compact` with test results summary |
| Reserve for corrections | 30K tokens (~15% of window) | If exceeded, `/clear` and reload from files |

Check your budget with `/cost` — it reports current token usage. When you hit the budget for a phase, stop and manage context before continuing.

**`--resume` checkpoint recovery steps:**

If a session was interrupted (crash, terminal closed, kill), recover with:

```bash
# Resume the most recent conversation:
claude -c

# If you need a specific older session, use the session selector:
claude --resume

# The selector shows a list of recent sessions with timestamps and
# the first line of each conversation. Pick the one you need.
```

---

### 4. Hook Error Debugging

Hooks fail in three common patterns. Each has a distinct diagnostic path. For Hook fundamentals and configuration format, see [03-1 Hooks Basics](../03-automation/03-1-hooks-basics.md). For advanced patterns, see [03-2 Hooks Advanced](../03-automation/03-2-hooks-advanced.md).

**Pattern 1: Infinite loop — Hook triggers itself**

Symptom: Claude Code becomes extremely slow or unresponsive after a tool call. CPU usage spikes. The terminal may show repeated output.

Cause: A `PostToolUse` Hook on `Edit` modifies a file, which triggers another `Edit`, which triggers the Hook again.

Example of the problematic config:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write \"$(echo $(cat) | jq -r '.tool_input.file_path')\""
          }
        ]
      }
    ]
  }
}
```

Fix: PostToolUse Hooks must not trigger additional tool calls. Use direct shell commands (like `npx prettier --write`) that modify files on the filesystem without going through Claude Code's tool system. The config above is actually safe because `npx prettier --write` runs outside the agent loop. The real danger is when a Hook's output instructs Claude to perform another edit. Ensure your Hook's stdout does not contain instructions like "please fix this file."

**Pattern 2: Matcher mismatch — Hook never triggers**

Symptom: You configured a Hook to protect `.env` files, but Claude edits `.env` without being blocked.

Diagnostic flow:

```bash
# Step 1: Check what hooks are loaded
# In Claude Code, run:
/hooks

# Step 2: Verify the matcher value matches the tool name exactly.
# Common mistake: using "edit" instead of "Edit" (case-sensitive)
# Common mistake: using "Bash" as matcher for a PreToolUse Hook
# that should match "Edit" or "Write"

# Step 3: Test the script manually with simulated input:
echo '{"tool_name":"Edit","tool_input":{"file_path":".env"}}' | bash .claude/hooks/protect-files.sh
echo $?
# Expected: exit code 2 and a stderr message
```

If the manual test produces exit code 0 instead of 2, the bug is in the script logic, not the matcher.

**Pattern 3: Script execution failure**

Symptom: Claude Code prints `Hook error: <event> hook failed` or the Hook silently does nothing.

Real error messages and their fixes:

| Error Message | Cause | Fix |
|---|---|---|
| `Hook error: command not found` | The script path in `command` is wrong, or the script is not executable | Verify the path; run `chmod +x` on the script |
| `Hook error: /bin/bash: .claude/hooks/check.sh: Permission denied` | Missing execute permission | `chmod +x .claude/hooks/check.sh` |
| `Hook error: jq: command not found` | `jq` is not installed or not in PATH | Install jq: `brew install jq` (macOS), `apt install jq` (Debian/Ubuntu) |
| `Hook error: unexpected end of JSON input` | The Hook script tries to parse stdin but receives empty input | Check that `cat` is reading stdin correctly; some events do not provide `tool_input` |
| No error shown, Hook silently ignored | The Hook's exit code is non-zero (but not 2), so the system treats it as a Hook error and continues | Add `; exit 0` at the end of commands that may return non-zero, or add `exit 2` explicitly for blocking |

**Complete diagnostic flow for any Hook failure:**

```bash
# 1. In Claude Code, list hooks:
/hooks

# 2. Identify the suspect hook. Copy its command.

# 3. In a separate terminal, simulate the hook input:
echo '{"session_id":"test","hook_event_name":"PreToolUse","tool_name":"Edit","tool_input":{"file_path":"src/index.ts","old_string":"x","new_string":"y"}}' | bash -c '<paste the command here>'

# 4. Check the exit code:
echo $?

# 5. Check stderr output (if blocking):
echo '{"session_id":"test","hook_event_name":"PreToolUse","tool_name":"Edit","tool_input":{"file_path":".env"}}' | bash -c '<paste the command here>' 2>&1

# 6. Fix the script, then test again. No need to restart Claude Code —
#    hooks are re-read on every invocation.
```

---

### 5. MCP Server Connection Failures

MCP servers extend Claude Code's capabilities with external tools. When they fail, the tools they provide become unavailable. For MCP server setup and configuration, see [03-4 MCP Servers](../03-automation/03-4-mcp-servers.md).

**Common causes and their diagnostics:**

| Cause | Symptom | Diagnostic Command | Fix |
|---|---|---|---|
| Token expired | `/mcp` shows server as "error" or "disconnected" | Check token expiry in config file; try a manual API call | Regenerate the token and update `.mcp.json` |
| Port conflict | Server fails to start; "address already in use" in logs | `lsof -i :<port>` (macOS/Linux) or `netstat -ano | findstr :<port>` (Windows) | Kill the conflicting process or change the port in config |
| Config format error | Server does not appear in `/mcp` output at all | Validate JSON: `cat .mcp.json | python3 -m json.tool` | Fix the JSON syntax error (missing comma, trailing comma, wrong nesting) |
| Server binary not found | `/mcp` shows "command not found" error | `which <server-binary>` or `npx <server-package> --version` | Install the server: `npm install -g <package>` or add to project devDependencies |
| Stdio server crashes on start | `/mcp` shows "disconnected" immediately after start | Run the server command manually in a terminal to see its error output | Fix the server's startup error (missing env vars, wrong arguments) |

**Diagnostic flow:**

```bash
# Step 1: Check MCP server status in Claude Code:
/mcp

# Step 2: If a server shows as disconnected, try restarting it from /mcp menu.

# Step 3: If restart fails, test the server manually.
# For HTTP-based MCP servers:
curl -X POST http://localhost:3100/health
# A healthy server responds with 200 OK.

# For stdio-based MCP servers, run the command from .mcp.json directly:
npx @anthropic/mcp-server-filesystem /path/to/allowed/dir
# Check if it starts without errors.

# Step 4: Validate the config file:
cat .mcp.json | python3 -m json.tool
# This will print a clear error if the JSON is malformed.

# Step 5: Check for environment variable issues:
# Some MCP servers require tokens or API keys in env vars.
# The .mcp.json "env" field passes these to the server process:
```

Example `.mcp.json` with environment variables:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "<your-token-here>"
      }
    }
  }
}
```

If the token is expired or revoked, the server starts but fails on every API call. The fix is to generate a new token and update the config.

---

### 6. Worktree Conflicts & Rollback

Git Worktree enables parallel Claude Code sessions, but creates its own class of problems. For worktree setup and parallel development workflow, see [04-3 Parallel Development & Worktree](../04-orchestration/04-3-worktree.md).

**Common Issue 1: Orphan worktree**

Symptom: `git worktree list` shows a worktree whose directory has been deleted.

```bash
$ git worktree list
/home/user/project                  abc1234 [main]
/home/user/project-feature-auth     def5678 [feature-auth]  # directory was deleted

$ git worktree prune
# Removes references to worktrees whose directories no longer exist.

$ git worktree list
/home/user/project                  abc1234 [main]
```

**Common Issue 2: Branch lock — "is already checked out"**

Symptom: You try to check out a branch that is already checked out in another worktree.

```bash
$ git checkout feature-auth
fatal: 'feature-auth' is already checked out at '/home/user/project-feature-auth'
```

Fix: Either work in the existing worktree, or remove it first:

```bash
# Option 1: Navigate to the existing worktree
cd /home/user/project-feature-auth

# Option 2: Remove the worktree if you are done with it
git worktree remove /home/user/project-feature-auth
# Then check out the branch normally.
```

**Common Issue 3: Merge conflicts across worktrees**

When two worktrees modify the same file on different branches, you get merge conflicts when merging. This is standard Git behavior, not a worktree-specific bug.

Resolution:

```bash
# Step 1: Merge the feature branch into main
git checkout main
git merge feature-auth

# Step 2: If conflicts arise, resolve them:
git status
# Shows files with conflicts marked as "both modified"

# Step 3: Edit each conflicted file, resolve the markers, then:
git add <resolved-file>
git commit
```

**Worktree cleanup commands reference:**

| Command | What It Does |
|---|---|
| `git worktree list` | Show all worktrees, their paths, and checked-out branches |
| `git worktree prune` | Remove references to worktrees whose directories were deleted |
| `git worktree remove <path>` | Remove a specific worktree (deletes the directory and unlocks the branch) |
| `git worktree add <path> <branch>` | Create a new worktree for an existing branch |
| `git worktree add <path> -b <new-branch>` | Create a new worktree with a new branch |

---

### 7. Checkpoints & --resume Complete Usage

Claude Code provides three mechanisms for recovering from interruptions or undoing changes. Each works differently.

**Mechanism 1: `Esc + Esc` — Checkpoint rollback menu**

Press `Esc` twice in the Claude Code terminal to open the rollback menu. This shows a list of checkpoints — snapshots taken automatically before each tool call that modifies files.

```
Checkpoint rollback menu:
  1. Before Edit: src/middleware/auth.ts (2 minutes ago)
  2. Before Write: src/middleware/auth.test.ts (5 minutes ago)
  3. Before Bash: npm test (7 minutes ago)

Select a checkpoint to roll back to, or press Esc to cancel.
```

Selecting a checkpoint reverts all file changes made after that point. The conversation history is preserved — only the filesystem changes are undone. Use this when Claude made a wrong edit and you want to go back to a known-good state without losing the conversation.

**Mechanism 2: `claude --resume` — Session selector**

```bash
claude --resume
```

This opens an interactive list of recent sessions:

```
Recent sessions:
  1. [2 hours ago] "Implement auth middleware for ai-dev-assistant"
  2. [5 hours ago] "Fix dashboard API endpoint"
  3. [yesterday]   "Add ESLint config and pre-commit hooks"

Select a session to resume:
```

Use this when you closed a session and want to pick it up later, or when you need to return to a specific session from days ago.

**Mechanism 3: `claude -c` — Resume most recent conversation**

```bash
claude -c
```

No menu, no selection — it immediately resumes the most recent conversation. Use this after a crash, a `kill`, or when you closed the terminal by accident.

**Comparison table:**

| Mechanism | What It Recovers | Use Case | Scope |
|---|---|---|---|
| `Esc + Esc` | File system state (rolls back file changes) | Claude made a bad edit; undo it without losing conversation | Current session, file changes only |
| `claude --resume` | A specific past session (conversation + context) | Return to a session you closed hours or days ago | Any recent session |
| `claude -c` | The most recent session (conversation + context) | Recover from crash, kill, or accidental terminal close | Most recent session only |

**When none of these work:**

If `claude -c` reports "no recent session found," the session data may have been lost (e.g., disk full, or the `~/.claude/` directory was cleared). In this case, you start fresh. This is why the Document & Clear pattern from [04-2 Context Management](../04-orchestration/04-2-context-management.md) is critical — if your progress is recorded in files, you can always rebuild the session from those files.

---

## Hands-On Example: Simulating a Hook Error & Debugging It

**Objective**: Create a deliberately broken Hook, observe its failure mode, diagnose the root cause, and fix it — building the muscle memory for real Hook debugging.

**Prerequisites**:
- Completed [03-1 Hooks Basics](../03-automation/03-1-hooks-basics.md) (understand Hook configuration format and exit codes)
- A project with `.claude/settings.json` already created
- `jq` installed on the system

**Steps**:

**Step 1**: Create a Hook script with a deliberate bug. Save the following as `.claude/hooks/check-todo.sh`:

```bash
#!/bin/bash
# Bug: the jq filter references a field that does not exist in PostToolUse events
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.content // empty')
# Correct field for Edit tool is .tool_input.new_string, not .tool_input.content

if echo "$FILE_PATH" | grep -q "TODO"; then
  echo "Warning: file contains TODO markers" >&2
  exit 2
fi

exit 0
```

**Step 2**: Set the script as executable:

```bash
chmod +x .claude/hooks/check-todo.sh
```

**Step 3**: Register the Hook in `.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "bash \"$CLAUDE_PROJECT_DIR/.claude/hooks/check-todo.sh\""
          }
        ]
      }
    ]
  }
}
```

**Step 4**: In Claude Code, ask Claude to edit any file and add a line containing `TODO`. For example: "Add a comment `// TODO: implement error handling` to `src/index.ts`."

Observe: The Hook does not block the edit, even though the new content contains "TODO." The Hook silently passes because `jq -r '.tool_input.content'` returns empty (the field does not exist), so `$FILE_PATH` is empty, and the `grep` does not match.

**Step 5**: Diagnose the failure. In a separate terminal, simulate the Hook input:

```bash
echo '{"session_id":"test","hook_event_name":"PostToolUse","tool_name":"Edit","tool_input":{"file_path":"src/index.ts","old_string":"// placeholder","new_string":"// TODO: implement error handling"}}' | bash .claude/hooks/check-todo.sh
echo $?
```

The exit code is `0` (allowed) — not the expected `2` (blocked). The `jq` filter `.tool_input.content` returns null because the correct field is `.tool_input.new_string`.

**Step 6**: Fix the script. Replace `.tool_input.content` with `.tool_input.new_string`:

```bash
#!/bin/bash
INPUT=$(cat)
NEW_CONTENT=$(echo "$INPUT" | jq -r '.tool_input.new_string // empty')

if echo "$NEW_CONTENT" | grep -q "TODO"; then
  echo "Warning: edit introduces a TODO marker. Remove or resolve the TODO before proceeding." >&2
  exit 2
fi

exit 0
```

**Step 7**: Test the fix manually:

```bash
echo '{"session_id":"test","hook_event_name":"PostToolUse","tool_name":"Edit","tool_input":{"file_path":"src/index.ts","old_string":"// placeholder","new_string":"// TODO: implement error handling"}}' | bash .claude/hooks/check-todo.sh
echo $?
```

The exit code is now `2`, and stderr shows the warning message.

**Step 8**: Return to Claude Code and repeat the edit request. The Hook now triggers and sends the warning message to Claude.

**Expected Results**:
- Step 4: Claude edits the file without any Hook interference (the bug causes silent passthrough)
- Step 5: Manual test confirms exit code `0` — the Hook logic is wrong, not the registration
- Step 7: After the fix, manual test confirms exit code `2` and stderr output: `Warning: edit introduces a TODO marker. Remove or resolve the TODO before proceeding.`
- Step 8: In Claude Code, the Hook fires and Claude receives the warning as context

**Common Errors**:

**Error 1: `bash: .claude/hooks/check-todo.sh: No such file or directory`**

Symptom: The manual test in Step 5 fails immediately with a "No such file" error.

Cause: You are running the command from a directory other than the project root. The relative path `.claude/hooks/check-todo.sh` only works from the project root.

Fix: Either `cd` to the project root before running the test, or use the absolute path: `bash /full/path/to/project/.claude/hooks/check-todo.sh`.

**Error 2: Exit code is `0` even after the fix**

Symptom: After updating the script in Step 6, the manual test still returns exit code `0`.

Cause: You edited the wrong file, or the file was not saved. Another common cause: the test input JSON has a typo in the `new_string` field (e.g., `"todo"` lowercase instead of `"TODO"` uppercase — `grep -q "TODO"` is case-sensitive).

Fix: Verify you are editing the correct file path. Run `cat .claude/hooks/check-todo.sh` to confirm the fix is saved. If the issue is case sensitivity, either use `grep -qi "TODO"` for case-insensitive matching, or ensure your test input matches the pattern exactly.

**Error 3: Hook triggers on PostToolUse but cannot actually block the edit**

Symptom: The Hook fires and Claude receives the warning, but the file was already modified because PostToolUse runs after the operation.

Cause: This is not a bug — it is how PostToolUse works. PostToolUse cannot prevent an operation; it can only report after the fact.

Fix: If you need to prevent edits containing TODO markers, move the Hook to `PreToolUse` instead of `PostToolUse`. PreToolUse runs before the tool executes and `exit 2` blocks the operation.

---

## Key Takeaways

- Always gather diagnostic information before acting: run `/cost`, `/hooks`, `/mcp`, or `git worktree list` before attempting a fix
- A frozen spinner with no status updates for 60+ seconds is a hang; a spinner with "Thinking..." is extended reasoning — do not kill a working process
- Context explosion is recoverable with `/compact` if the task is in progress; use `/clear` only when you can afford to lose the conversation state
- Context budget allocation (planning your token spend per phase) prevents context explosions from occurring in the first place
- Hook debugging follows a fixed sequence: `/hooks` to verify registration, manual script execution with simulated JSON to isolate the bug, check exit code to confirm the fix
- `Esc+Esc` rolls back file changes; `claude -c` resumes the session — these solve different problems and are not interchangeable

---

## Self-Assessment

1. You are in the middle of implementing a feature. Claude starts forgetting instructions you gave 10 minutes ago, and its responses are getting shorter. `/cost` shows 75% context usage. What is your recovery sequence? What information do you pass to `/compact`?

2. You configured a `PreToolUse` Hook to block writes to `*.config.js` files, but Claude just overwrote `jest.config.js` without being blocked. Describe the exact diagnostic steps you would take, starting from `/hooks` and ending with a confirmed fix.

3. After your laptop crashed during a Claude Code session, you reopen the terminal. The feature implementation was halfway done. What command do you run first? What do you do if that command reports "no recent session found"?

4. Two developers on the same repo are using `claude --worktree` for different features. Developer A finishes and tries to merge, but gets merge conflicts because Developer B modified the same files. What Git commands resolve this, and how do they prevent it next time?


---

⬅️ [Previous: Resource Index](08-2-resources.md) ｜ 📖 [Index](../00-index.md)

🌐 [繁體中文版](../../zh/08-參考資源/08-3-災難復原與Debug指南.md)
