# 02-2 Settings & Permissions
> Use settings.json to predefine operational boundaries, allowing Claude Code to work autonomously within a controlled scope.

## Learning Objectives
- Understand the 6-layer priority structure of Claude Code settings
- Master the allow / deny rule syntax in settings.json
- Be able to configure reasonable permission boundaries for a project
- Understand the differences and use cases of the five permission modes

---

## Why Settings and Permissions Matter

Without settings, Claude Code prompts for permission on every operation. You end up spending a significant amount of time clicking "Allow." Worse, Claude might execute actions you do not want it to perform—deleting files, modifying `.env`, or force pushing. The purpose of the settings and permissions system is to predefine what is allowed and what is not, reducing manual intervention while protecting critical resources from accidental operations in automated workflows.

A properly configured work environment lets Claude run tests, format code, and commit to git without ever interrupting you, while preventing it from touching `.env` or executing destructive git commands. This is the balance between "letting AI work efficiently" and "keeping humans in control."

---

## Layered Priority

Claude Code settings have 6 layers, from highest to lowest priority:

| Priority | Source | Description |
|----------|--------|-------------|
| Highest | Managed policy | Set uniformly by enterprise administrators; cannot be overridden by users |
| ↓ | CLI flags (--flag) | Effective for the current session only; expires when the session ends |
| ↓ | .claude/settings.local.json | Local project settings (should be added to gitignore) |
| ↓ | .claude/settings.json | Shared project settings (committed to git) |
| ↓ | ~/.claude/settings.json | User-level global settings |
| Lowest | Plugin configuration | Default settings provided by plugins |

In practice, the most commonly used layers are `.claude/settings.json` (shared baseline rules for the team) and `.claude/settings.local.json` (personal overrides, not committed to version control). The former defines "what the project allows," while the latter defines "my personal environment variables or preferences."

---

## Key Fields in settings.json

Below is the most practical settings structure. You do not need to memorize every field—just understand this core framework:

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Bash(npx jest *)",
      "Bash(git commit *)",
      "Bash(git push *)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)",
      "Edit(.env*)"
    ]
  },
  "env": {
    "NODE_ENV": "development"
  }
}
```

The `allow` array lists operations that are "pre-approved and require no confirmation." Whenever Claude attempts to execute these commands, it proceeds directly without displaying a prompt.

The `deny` array lists operations that are "explicitly forbidden and will be refused even if requested." This is the security baseline and is not affected by other settings.

The `env` block injects environment variables so that Claude has the correct execution environment when running commands.

---

## Five Permission Modes

In addition to static allow / deny rules, Claude Code has five overall modes that control how operations not explicitly listed are handled:

| Mode | Behavior | Use Case |
|------|----------|----------|
| default | Prompts for confirmation on every tool call | Standard daily use |
| plan | Read-only exploration; no modifications allowed | Analyzing a codebase, understanding existing architecture |
| acceptEdits | Automatically accepts all file edits | Well-defined tasks (e.g., refactoring a specific module) |
| dontAsk | Automatically rejects tools not listed in allow | Scoped automation tasks |
| bypassPermissions | Skips all prompts | Use only in isolated sandbox environments |

To switch modes: press `Shift+Tab` in the conversation interface to cycle through them, or type `/permissions` to select one.

The `bypassPermissions` mode is dangerous and should not be used in local development environments. Even in this mode, Claude Code retains confirmation prompts for the following directories: `.git/`, `.claude/`, `.vscode/`, `.idea/`. These are protected directories and are not affected by mode settings.

---

## allow / deny Rule Syntax

The rule syntax follows the format "ToolName(parameter pattern)" and supports the following four matching methods:

| Pattern | Example | Match Description |
|---------|---------|-------------------|
| Exact match | `Bash(npm run test)` | Matches only the exact same command |
| Prefix wildcard | `Bash(npm *)` | All commands starting with npm |
| Path pattern | `Edit(.env*)` | .env, .env.local, .env.production, etc. |
| MCP tool | `mcp__github__*` | All tools provided by the GitHub MCP |

There is an important principle when designing rules: allow can be broad, but deny should be precise.

A broad allow (e.g., `Bash(npm run *)`) has the downside of occasionally permitting unexpected npm scripts, but this is usually acceptable. If deny is too broad (e.g., `Bash(git *)`), it blocks safe operations like `git status` and `git log` as well, causing unnecessary friction. Deny should target only destructive operations, such as `Bash(git push --force *)` and `Bash(git reset --hard *)`, rather than the entire git toolset.

---

## Hands-On Example: Configuring settings.json for the AI Dev Assistant

> Main case continued: Establish a complete permission boundary configuration in the ai-dev-assistant project so that Claude can autonomously handle development tasks while protecting critical resources.

**Goal**: Create `.claude/settings.json` to allow routine development operations and block dangerous commands; also create a personal local settings file `.claude/settings.local.json`.

**Prerequisites**: Completed 01-2 (project scaffolding) and 02-1 (CLAUDE.md configuration); `package.json` and `.gitignore` exist in the project root directory.

**Steps**:

1. Create the `.claude` directory in the project root:
   ```bash
   mkdir -p .claude
   ```

2. Create `.claude/settings.json` with the following content:
   ```json
   {
     "permissions": {
       "allow": [
         "Bash(npm run *)",
         "Bash(npx jest *)",
         "Bash(npx tsc *)",
         "Bash(npx prettier *)",
         "Bash(npx eslint *)",
         "Bash(git add *)",
         "Bash(git commit *)",
         "Bash(git push *)",
         "Bash(git status)",
         "Bash(git diff *)",
         "Bash(git log *)"
       ],
       "deny": [
         "Bash(rm -rf *)",
         "Bash(git push --force *)",
         "Bash(git reset --hard *)",
         "Edit(.env*)",
         "Edit(.claude/settings.json)"
       ]
     }
   }
   ```

3. Verify that the JSON syntax is correct (run immediately after writing):
   ```bash
   node -e "JSON.parse(require('fs').readFileSync('.claude/settings.json','utf8')); console.log('JSON valid')"
   ```

4. Create `.claude/settings.local.json` (personal settings, not committed to git):
   ```json
   {
     "env": {
       "NODE_ENV": "development"
     }
   }
   ```

5. Add the local settings to `.gitignore`. Open `.gitignore` and add:
   ```
   .claude/settings.local.json
   ```

6. Start Claude Code and try running `npm test`—Claude should execute it directly without displaying a confirmation prompt.

7. Ask Claude to modify `.env` in the conversation—Claude should report that the operation was denied and not perform the modification.

**Expected Results**:
- Operations such as `npm run *` and `git add / commit / push / status / diff / log` no longer require manual confirmation
- Attempts to execute `rm -rf`, `git push --force`, or `git reset --hard` are rejected by the system
- Attempts to edit `.env` or `.env.local` are rejected by the system
- `.claude/settings.local.json` appears in `.gitignore` and is not tracked by `git status`

**Common Mistakes**:

- **JSON syntax errors (missing or trailing commas)**: Claude Code reports a `parse error` on startup and cannot load the settings. Validate using the node command above, or install `jq` and run `jq . .claude/settings.json`.

- **Deny rules that are too broad**: For example, writing `Bash(git *)` blocks `git status` and `git log` entirely, preventing Claude from viewing git status. Deny should target specific dangerous operations; do not use overly broad wildcards.

- **Forgetting to add settings.local.json to .gitignore**: Personal environment variables or API keys may leak into version control. Every time you create a local settings file, immediately verify that `.gitignore` has been updated.

- **Allow rules are defined but confirmation prompts still appear**: Verify that the rule syntax is correct—tool names are case-sensitive (`Bash` not `bash`), and the parameter pattern must exactly match the actual command prefix.

---

## Key Takeaways

- settings.json has 6 priority layers; `.claude/settings.json` (shared) and `.claude/settings.local.json` (personal) are the two most commonly used layers in daily work
- Allow rules let pre-approved operations skip confirmation; deny rules establish an uncrossable security baseline
- Deny should be precise (targeting dangerous operations); allow can be broad (permitting entire categories of operations)
- Even in bypassPermissions mode, `.git/` and `.claude/` remain protected directories
- settings.local.json must be added to .gitignore to prevent personal environment settings or sensitive information from entering version control

---

## Security Best Practices

The allow/deny rules and five permission modes covered above are "mechanisms." This section focuses on "strategy" — how to combine these mechanisms in different scenarios to prevent the Agent from accessing things it shouldn't.

### Dangerous Operations Checklist

The following operations should be added to deny rules, ordered by risk level:

| Risk Level | Deny Rule | Protection Target |
|-----------|-----------|-------------------|
| 🔴 Critical | `Bash(rm -rf *)` | Prevent recursive deletion of entire directories |
| 🔴 Critical | `Bash(git push --force *)` | Prevent overwriting remote history |
| 🔴 Critical | `Bash(git reset --hard *)` | Prevent discarding uncommitted changes |
| 🔴 Critical | `Edit(.env*)` | Prevent modification of environment variables and secrets |
| 🟠 High | `Edit(*.pem)` | Prevent modification of SSL certificates and private keys |
| 🟠 High | `Edit(*secret*)` | Prevent modification of files containing secrets |
| 🟠 High | `Edit(*credential*)` | Prevent modification of credential files |
| 🟠 High | `Bash(chmod 777 *)` | Prevent opening all file permissions |
| 🟡 Medium | `Bash(curl * \| sh)` | Prevent downloading and executing unknown scripts |
| 🟡 Medium | `Edit(.claude/settings.json)` | Prevent Agent from modifying its own permissions |

Actual settings.json configuration example:

```json
{
  "permissions": {
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)",
      "Bash(git reset --hard *)",
      "Edit(.env*)",
      "Edit(*.pem)",
      "Edit(*secret*)",
      "Edit(*credential*)",
      "Edit(.claude/settings.json)"
    ]
  }
}
```

### Sensitive Information Protection

Claude Code reads files within your project during task execution. If `.env` or credential files are not protected by deny rules, the Agent may read their contents during debugging or analysis, and potentially write sensitive information into commit messages, PR descriptions, or log output.

**File patterns that must be denied**:

```json
{
  "permissions": {
    "deny": [
      "Read(.env*)",
      "Edit(.env*)",
      "Read(**/credentials.json)",
      "Read(**/*token*)",
      "Read(**/*secret*)"
    ]
  }
}
```

**CI/CD environment distinction**: In CI/CD workflows, environment variables are injected through the runner's env (e.g., GitHub Actions' `${{ secrets.API_KEY }}`), not hardcoded in settings.json. The settings.json should only contain permission rules, never actual secret values.

### Team Permission Minimization

When collaborating in teams, the core permission strategy is: **shared settings guard the baseline, personal settings add flexibility**.

| Settings Layer | File | Strategy | Version Controlled |
|---------------|------|----------|-------------------|
| Team shared | `.claude/settings.json` | Only deny rules (security baseline) | ✅ Yes |
| Personal override | `.claude/settings.local.json` | Allow rules (personal preferences) | ❌ No |

**Principles**:

1. **New team members use default permission mode**. Onboarding docs should never instruct new members to set `bypassPermissions` — this skips all safety confirmations
2. **Deny rules are maintained by the Tech Lead**, committed to version control via `.claude/settings.json`, automatically inherited by all team members
3. **Personal allow expansions don't override team baselines**. Even if someone adds broader allow rules in their local settings, team-level deny rules still take effect (deny has higher priority than allow)
4. **Audit permission configurations periodically**. Use the `/permissions` command to check currently active rules and confirm there are no outdated or overly broad allows

---

## Self-Assessment

- If you want Claude to be able to run `npx jest --coverage` but do not want to allow all `npx` commands, how should you write the allow rule?
- When both `.claude/settings.json` and `~/.claude/settings.json` exist and have conflicting settings, which one takes effect?
- Why is it recommended to add `Edit(.claude/settings.json)` to the deny rules?
- A team member writes `"allow": ["Edit(.env)"]` in `.claude/settings.local.json`, but the team's `.claude/settings.json` has `"deny": ["Edit(.env*)"]`. Can Claude edit `.env`? Why or why not?
- In a CI/CD environment, why shouldn't you put API keys in settings.json? What is the correct approach?

---

⬅️ [Previous: CLAUDE.md Complete Guide](02-1-claude-md-guide.md) ｜ 📖 [Index](../00-index.md) ｜ ➡️ [Next: Agent = Model + Harness](02-3-harness-preview.md)

🌐 [繁體中文版](../../zh/02-基礎配置/02-2-設定檔與權限系統.md)
