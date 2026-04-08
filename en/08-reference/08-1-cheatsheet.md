# 08-1 Cheatsheet

## Table 1: Slash Commands

| Command | Function | Usage Frequency |
|---------|----------|-----------------|
| `/help` | Show all available commands | High |
| `/init` | Generate CLAUDE.md based on the project | High |
| `/plan` | Toggle Plan Mode | High |
| `/clear` | Reset context | High |
| `/compact [instructions]` | Compress conversation history | High |
| `/model` | Switch model | Medium |
| `/cost` | View spending | Medium |
| `/memory` | View/edit memory | Medium |
| `/permissions` | Manage permissions | Medium |
| `/hooks` | Browse configured Hooks | Medium |
| `/mcp` | Manage MCP servers | Medium |
| `/agents` | Manage Subagents | Medium |
| `/review` | Code review | Medium |
| `/btw` | Side question (not saved to history) | Medium |
| `/simplify` | Simplify review | Medium |
| `/batch` | Large-scale parallel changes | Low |
| `/loop` | Loop execution | Low |
| `/debug` | Debug session | Low |
| `/rename` | Rename session | Low |
| `/export` | Export conversation | Low |
| `/vim` | Enable Vim mode | Low |
| `/terminal-setup` | Set up terminal shortcuts | Low |

---

## Table 2: Keyboard Shortcuts

### Terminal Mode

| Shortcut | Function |
|----------|----------|
| `Ctrl+C` | Cancel current operation |
| `Ctrl+D` | Exit session |
| `Ctrl+L` | Clear terminal |
| `Ctrl+O` | Toggle verbose output |
| `Ctrl+R` | Reverse history search |
| `Ctrl+V` / `Cmd+V` | Paste image from clipboard |
| `Ctrl+B` | Run task in background |
| `Ctrl+T` | Toggle task list |
| `Ctrl+G` | Edit prompt in editor |
| `Esc + Esc` | Rollback menu (Checkpoint) |
| `Shift+Tab` | Toggle permission mode |
| `Alt+P` | Switch model |
| `Alt+T` | Toggle extended thinking |

### Text Editing

| Shortcut | Function |
|----------|----------|
| `Ctrl+K` | Delete to end of line |
| `Ctrl+U` | Delete entire line |
| `Ctrl+Y` | Paste deleted text |
| `Alt+B` / `Alt+F` | Word navigation |

---

## Table 3: CLI Command Reference

| Command | Description |
|---------|-------------|
| `claude` | Start an interactive session |
| `claude "prompt"` | Start with an initial prompt |
| `claude -p "query"` | Non-interactive query |
| `claude -c` | Continue the most recent conversation |
| `claude -r "name"` | Resume a specific session |
| `claude --resume` | Show session selector |
| `claude --worktree name` | Worktree isolation |
| `claude --agent name` | Use a specific Subagent |
| `claude --model opus` | Specify model |
| `claude --permission-mode plan` | Specify permission mode |
| `claude --version` | Show version |
| `claude doctor` | Diagnostic check |
| `claude update` | Update |
| `claude auth login` | Log in |
| `claude auth logout` | Log out |
| `claude mcp add` | Add MCP server |
| `claude mcp list` | List MCP servers |

---

## Table 4: Hook Event Quick Reference

| Event | Trigger Timing | Blockable | Common Use Cases |
|-------|---------------|-----------|------------------|
| `SessionStart` | Session starts/resumes | No | Load context |
| `UserPromptSubmit` | Before submitting a prompt | Yes | Input preprocessing |
| `PreToolUse` | Before tool execution | Yes | Protect files, validate commands |
| `PostToolUse` | After tool execution | No | Formatting, logging |
| `PostToolUseFailure` | After tool failure | No | Error handling |
| `PermissionRequest` | Before permission prompt | Yes | Auto-approve |
| `Stop` | Claude finishes response | Yes | Validator Pattern |
| `SubagentStart` | Subagent starts | No | Tracking |
| `SubagentStop` | Subagent finishes | No | Notification |
| `PreCompact` | Before compaction | No | Backup |
| `PostCompact` | After compaction | No | Re-injection |
| `ConfigChange` | Configuration changes | No | Auditing |

---

## Table 5: Permission Syntax Quick Reference

| Pattern | Example | Description |
|---------|---------|-------------|
| Exact match | `Bash(npm run test)` | Matches this command only |
| Prefix wildcard | `Bash(npm *)` | All commands starting with npm |
| Suffix wildcard | `Bash(* test)` | All commands ending with test |
| Path pattern | `Edit(.env*)` | Files starting with .env |
| MCP tool | `mcp__github__*` | All GitHub MCP tools |
| Subagent | `Agent(Explore)` | Specific Subagent |

---

## Table 6: Common Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| `command not found: claude` | Not installed or not in PATH | Reinstall, check PATH |
| `You need to authenticate` | Not logged in | `claude auth login` |
| Claude ignores CLAUDE.md rules | File too long or rules too vague | Trim to 200 lines, use emphatic language |
| Hook not triggering | Typo in matcher | Verify with `/hooks`, check case sensitivity |
| Hook infinite loop | Stop Hook not checking `stop_hook_active` | Add `stop_hook_active` check |
| MCP connection failed | Invalid token or malformed config | Check status with `/mcp` |
| Context overload | Session too long | `/clear` or `/compact` |
| JSON parse error | Syntax error in settings.json | Validate with `node -e "JSON.parse(...)"` |

---

⬅️ [Previous: David Chu Evolution Framework](../07-frameworks/07-4-david-chu.md) ｜ 📖 [Index](../00-index.md) ｜ ➡️ [Next: Resource Index](08-2-resources.md)

🌐 [繁體中文版](../../zh/08-參考資源/08-1-速查表.md)
