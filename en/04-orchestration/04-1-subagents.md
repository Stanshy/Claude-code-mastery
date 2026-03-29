# 04-1 Custom Subagents
> Separate complex roles from the main conversation, allowing Claude Code to execute dedicated tasks with a clean context.

## Learning Objectives
- Understand the core value of Subagents: independent context, with concise results returned to the main conversation
- Distinguish between built-in Subagents and custom Subagents and their applicable scenarios
- Master the key frontmatter fields and writing guidelines for `.claude/agents/<name>.md`
- Choose the correct invocation method (auto-delegation, @-mention, session-wide)

---

## Content

### 1. Why You Need Subagents

The main conversation's context is a limited resource — and the most easily overlooked one.

When you ask Claude Code to investigate a complex issue, the process generates a large amount of intermediate artifacts: which files were read, tool call outputs, directions attempted but failed, and reasoning chains that gradually narrow down the scope. These intermediate artifacts are valuable during the investigation, but are almost meaningless to the main conversation — all you truly need is the final answer.

The core value of Subagents is precisely cutting off this pollution path: a Subagent executes the complete investigation process in its own independent context and returns only a concise result to the main conversation. The main conversation's context remains unaffected and can continue handling subsequent tasks.

**David Chu's Graduation Path** illustrates the evolution of role definitions: beginners start by writing role descriptions in `CLAUDE.md`, then advance to Skills (repeatable workflows) as they gain familiarity, and ultimately build custom Agents (persistent roles with a defined persona). The distinction between the two lies here: Skills are suited for "single repeatable workflows" (e.g., deployment, formatting), while Agents are suited for "the same role needed across multiple types of tasks" (e.g., security reviewer, debugging specialist). Define tasks for Skills; define personas for Agents.

---

### 2. Built-in vs Custom Subagents

Claude Code has three built-in Subagents, each with clear usage boundaries:

| Subagent | Model | Tools | Primary Use |
|---|---|---|---|
| Explore | claude-haiku | Read-only (Read, Grep, Glob) | Quick codebase search, low cost |
| Plan | Inherited from main conversation | Read-only | Requirements research in planning mode |
| general-purpose | Inherited from main conversation | All tools | Complex multi-step tasks |
| Custom | Configurable | Configurable | Dedicated roles with specific personas |

Built-in Subagents are designed to be general-purpose. When your project needs a "security reviewer who uses the same set of standards every time it reviews the auth module," or a "debugging specialist dedicated to isolating and fixing bugs," built-in Subagents cannot carry such personas and constraints — this is the starting point for custom Subagents.

---

### 3. Creating Custom Subagents: File Structure and Field Reference

A custom Subagent is a Markdown file placed in the `.claude/agents/` directory. The filename serves as the Agent's identifier (lowercase, using hyphens).

**Standard File Location**

```
.claude/
└── agents/
    ├── security-reviewer.md
    └── debugger.md
```

**Key Frontmatter Fields**

```markdown
---
name: security-reviewer
description: Reviews code for security vulnerabilities. Use after any changes to authentication, authorization, or data handling code.
tools: Read, Grep, Glob
model: claude-sonnet-4-5
memory: project
isolation: worktree
---
```

Purpose and considerations for each field:

- **`name`**: Identifier used for @-mention invocation. Must match the filename, using lowercase hyphenated format.
- **`description`**: This is the most important field. Claude Code uses the description to decide whether to automatically delegate a task to this Agent. The description must clearly state the "trigger conditions," such as "Use after any changes to authentication," rather than a vague "reviews security."
- **`tools`**: List of allowed tools. Precisely restricting tools is the foundation of security — a security reviewer only needs read permissions and should not have Edit or Bash.
- **`model`**: Specifies the model to use. Use `claude-sonnet-4-5` or `claude-opus-4-5` for high-precision tasks (security review, architectural decisions); use `claude-haiku-4-5` for high-frequency, low-complexity tasks to save costs. If omitted, inherits the main conversation's model.
- **`memory: project`**: Enables persistent memory, allowing the Agent to remember project-specific knowledge across sessions.
- **`isolation: worktree`**: Runs the Agent in an isolated git worktree to prevent modifications from affecting the main working directory (suitable for Agents like debugger that need to experiment).

**System Prompt After the Frontmatter**

The Markdown content below the frontmatter serves as this Agent's system prompt. This is where you define the role's thinking framework, review criteria, and output format. The guiding principle when writing: be specific, not abstract. "Review code for SQL injection, XSS, command injection" produces more consistent work from the Agent than "Review code for security issues."

---

### 4. Three Invocation Methods

Once a custom Subagent is created, there are three ways to invoke it:

**Auto-delegation (Recommended for Daily Use)**

When you input a task to the main conversation, Claude Code scans the descriptions of all Agents to determine whether it should auto-delegate. For example, if you say "Help me confirm there are no security issues with this PR's auth module," Claude Code sees that the security-reviewer's description includes "authentication" and will auto-delegate. This is the method that requires the least additional effort, but it depends on the quality of the description.

**@-mention (When You Need to Specify Explicitly)**

```
@security-reviewer Check the src/auth/ directory
@debugger This test is failing, help me find the cause
```

Use this when you know exactly which Agent you want, or when auto-delegation doesn't trigger.

**Session-wide (Use the Same Agent for an Entire Session)**

```bash
claude --agent security-reviewer
```

Specified when launching from the command line, suitable when an entire session is dedicated to the same type of task (e.g., spending an entire afternoon on security reviews).

---

### 5. Skill vs Agent: Decision Table

| Requirement | Use Skill | Use Agent |
|---|---|---|
| Single fixed workflow (e.g., deployment, formatting) | ✓ | |
| Persistent role across multiple types of tasks (e.g., reviewer) | | ✓ |
| Independent context needed to avoid polluting the main conversation | | ✓ |
| Need to specify a particular model (e.g., haiku to reduce costs) | | ✓ |
| Need to precisely restrict tool permissions | | ✓ |
| Need git worktree isolation for an experimental environment | | ✓ |
| One-off quick task | ✓ | |

The essence of a Skill is "templatizing a fixed prompt"; the essence of an Agent is "giving a persistent role specific tools, a model, and a behavioral framework." If you find yourself repeatedly writing the same role description, that's the signal to create an Agent.

---

## Hands-on Example: Building security-reviewer and debugger Agents
> Continuing the main project: In the AI Dev Assistant project, build two dedicated Agents responsible for security review and debug isolation, respectively.

**Goal**: Create security-reviewer and debugger Agents for the ai-dev-assistant project, and verify that invocation behavior and tool restrictions work correctly.

**Prerequisites**: Completed 03-3 (Skills). The ai-dev-assistant project directory structure exists, including the `src/` directory.

**Steps**:

**Step 1**: Create the agents directory

```bash
mkdir -p .claude/agents
```

**Step 2**: Create `.claude/agents/security-reviewer.md`

```markdown
---
name: security-reviewer
description: Reviews code for security vulnerabilities. Use after any changes to authentication, authorization, API key handling, or data handling code. Trigger when asked to check security, review auth, or audit access control.
tools: Read, Grep, Glob
model: claude-sonnet-4-5
---

You are a senior security engineer with expertise in application security.

Review code for the following categories of vulnerabilities:
- Injection attacks: SQL injection, command injection, XSS
- Secrets exposure: hardcoded API keys, passwords, tokens in source code
- Authentication flaws: missing auth checks, broken session management
- Authorization flaws: privilege escalation, missing role checks
- Insecure data handling: logging sensitive data, unencrypted storage

For each finding, report in this exact format:
- **File**: path and line number
- **Severity**: critical / high / medium / low
- **Vulnerability**: one-line description
- **Evidence**: the specific code that is problematic
- **Fix**: concrete recommended change

If no issues are found, state "No vulnerabilities found" with the files reviewed.
```

**Step 3**: Create `.claude/agents/debugger.md`

```markdown
---
name: debugger
description: Isolates and fixes bugs in the codebase. Use when a specific bug needs focused investigation without polluting the main conversation context. Trigger when a test is failing, an error is thrown, or behavior is unexpected.
tools: Read, Edit, Write, Bash, Grep, Glob
model: claude-sonnet-4-5
isolation: worktree
---

You are a senior debugging specialist. Follow this exact process for every bug:

1. **Reproduce**: Read the error message or failing test output in full. Run the failing test to confirm the error.
2. **Isolate**: Trace the call stack. Identify the exact line where the incorrect behavior originates.
3. **Hypothesize**: State your hypothesis about the root cause before making any changes.
4. **Fix**: Implement the minimal change that addresses the root cause. Do not refactor unrelated code.
5. **Verify**: Run the failing test again to confirm it passes. Run the full test suite to confirm no regressions.
6. **Clean up**: Remove any temporary logging or debugging statements you added.

Report back to the main conversation with: root cause found, fix applied, tests passing.
```

**Step 4**: Launch Claude Code

```bash
claude
```

**Step 5**: Confirm the Agents are recognized

In the main conversation, enter:

```
List all available agents
```

You should see security-reviewer and debugger appear in the list.

**Step 6**: Test the security-reviewer's @-mention and tool restrictions

```
@security-reviewer Check the src/ directory for security vulnerabilities
```

Observe: security-reviewer only uses Read, Grep, and Glob, and does not attempt any modification operations. The main conversation only receives the review report and does not see the process of the Agent reading each file.

**Step 7**: Test the debugger's @-mention

First, introduce a known issue in `src/reviewer.ts`, or use an existing failing test, then enter:

```
@debugger Investigate the test failure in src/reviewer.test.ts
```

Observe: the debugger's investigation process (reading files, running tests, tracing the call stack) takes place in an independent context, and the main conversation only receives a concise summary of "root cause, fix applied, tests passing."

**Step 8**: Verify auto-delegation

Without using @-mention, enter directly:

```
Help me confirm that the auth token validation logic I just added has no security issues
```

Observe whether Claude Code automatically selects security-reviewer instead of handling it directly. If it does not auto-select, go back and check whether the trigger keywords in the description are sufficiently explicit.

**Expected Results**:
- Both Agents can be correctly triggered via @-mention
- The security-reviewer's tool restrictions are enforced: it only uses read-type tools and cannot execute Bash or modify files
- The debugger can read, modify files, and execute `npm test`
- The main conversation context is not bloated by the Agent's investigation process and remains concise throughout

**Common Mistakes**:

**Mistake 1: Agent does not respond to @-mention, or auto-delegation does not trigger**

Cause: The description field is too vague for Claude Code to determine the trigger conditions.

Bad example: `description: Reviews security issues`

Correct approach: Explicitly state trigger conditions and keywords in the description, for example: `description: Reviews code for security vulnerabilities. Use after any changes to authentication, authorization, API key handling, or data handling code. Trigger when asked to check security, review auth, or audit access control.`

**Mistake 2: Agent attempts to use a tool not listed in the tools field, resulting in a rejected operation**

Cause: The system prompt asks the Agent to perform operations that exceed the tools authorized in the tools field. For example, the security-reviewer's system prompt requires running `npm audit`, but the tools field does not include Bash.

Correct approach: Before writing the system prompt, finalize the tools field list first, and only describe operations in the system prompt that these tools can accomplish.

---

## Key Takeaways
- The fundamental value of Subagents is "independent context": complex investigation processes stay within the Subagent, and the main conversation only receives concise results
- The `description` field determines whether auto-delegation triggers; it must include explicit trigger conditions and keywords, not vague functional descriptions
- The `tools` field is a whitelist of tools — any tool not listed cannot be used, which is the concrete practice of the principle of least privilege
- Skills correspond to "repeatable workflows," while Agents correspond to "tasks requiring persistent roles and independent context" — the two are not mutually exclusive
- `isolation: worktree` allows Agents that need to experiment or modify files to run in an isolated git environment, protecting the main working directory

## Self-Assessment
- If @security-reviewer is not triggered by auto-delegation, what is your first troubleshooting step?
- The security-reviewer's `tools` field is set to `Read, Grep, Glob`, but you want it to also run `npm audit`. How should you adjust this while maintaining the principle of least privilege?
- In what situation would you choose to create an Agent rather than a Skill? Provide a specific scenario from the ai-dev-assistant project.
