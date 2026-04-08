# 01-1 What is Claude Code

> Claude Code is an autonomous execution agent that runs in the terminal, not a chatbot or an IDE plugin.

---

## Learning Objectives

After completing this chapter, you should be able to:

- Define Claude Code and its type precisely in one sentence
- Describe the four stages of the Agent Loop and its fundamental difference from "Q&A-style AI"
- Determine whether Claude Code is the right tool based on a given scenario
- Execute an Agent Loop hands-on and observe Claude Code autonomously completing a task

---

## Main Content

### 1. One-Sentence Definition

Claude Code is an **agentic coding CLI** (autonomous agent command-line tool) released by Anthropic.

There are three keywords in this sentence:

- **agentic**: It does not wait for you to direct it step by step; it proactively makes decisions, explores, and self-corrects
- **coding**: Its core capability is handling code and software engineering tasks
- **CLI**: It runs in the terminal and is not tied to any specific IDE

It can do the following four things, all autonomously: read the entire codebase, edit across multiple files, execute system commands (such as tests or builds), and verify whether its own changes are correct.

This is fundamentally different in approach from IDE plugins (like GitHub Copilot) that "complete the next line," or ChatGPT that "generates code snippets for you to paste."

---

### 2. Agent Loop

The working model of Claude Code is a continuously running loop, not a one-shot response:

```
Read → Decide → Execute → Verify
 ↑                           |
 └───────────────────────────┘
       (until task is complete)
```

**Read**: Claude Code scans the project structure, reads relevant files, and understands your requirements.
**Decide**: Determines what to do next, such as which file to modify or which command to invoke.
**Execute**: Actually performs file edits, runs tests, and invokes shell commands.
**Verify**: Confirms whether the results match expectations; if not, re-enters the loop.

This loop keeps running until the task is complete, or until it reaches a decision point that requires human input (for example: whether to delete a file you have not explicitly authorized for deletion).

The significance of this design is: you do not need to break a task into individual steps and hand them to Claude one by one. You can describe the end goal, and Claude plans the path and executes it through to completion. The trade-off is: you need to understand what it is doing, otherwise it is difficult to tell whether it has gone off track.

---

### 3. Differences from Other AI Coding Tools

The following comparison is not meant to assign scores, but to help you determine which tool to choose in a given situation (based on publicly available plans as of 2026-03; specifications and pricing may change):

| Dimension | Claude Code | Cursor | GitHub Copilot | Windsurf |
|-----------|-------------|--------|----------------|----------|
| Type | Terminal CLI | IDE (VS Code fork) | IDE Plugin | IDE |
| Working Mode | Autonomous agent loop | Agentic + Completion | Completion + Chat | Flow Mode |
| Multi-file Operations | Automatic cross-file | Automatic cross-file | Requires manual selection | Automatic cross-file |
| Command Execution | Can run tests, deploy | Can run commands | Cannot | Can run commands |
| Context Window | 200K (maximum) | 128K | Depends on model | 128K |
| Strongest Scenario | Complex architecture, autonomous multi-step tasks | Day-to-day IDE coding | Boilerplate completion | Intent-inference workflows |
| Starting Price | $20/mo (Pro) | $20/mo (Pro) | $10/mo | $15/mo |

Claude Code's 200K context window means it can understand a larger codebase at once, without requiring you to manually select "relevant files." This makes a noticeable difference when working on medium to large projects.

---

### 4. When to Use and When Not to Use

**Claude Code is best suited for the following scenarios:**

- **Building new features**: Describe your requirements, and Claude autonomously plans the implementation path and completes it
- **Fixing bugs**: Paste the error message, and Claude locates the root cause, fixes it, and runs tests to verify
- **Code review and refactoring**: Specify a file or scope, and Claude analyzes it and proposes changes
- **Writing tests**: Automatically generates corresponding test cases based on existing code
- **Git workflows**: Commit message generation, PR descriptions, branch management
- **Documentation maintenance**: Updates README or API docs based on the current state of the code

**The following scenarios are not recommended for Claude Code:**

- **Real-time autocomplete**: When you need character-by-character completion as you type, Copilot's latency and integration are better suited for this need
- **Visual UI development**: When you need to frequently preview results in a browser and adjust styles in real time, a CLI tool lacks this feedback loop
- **Extremely small single-line changes**: If you just need to rename a variable, the overhead of starting an Agent Loop is not worth it

Guiding principle: if a task requires "multiple steps, multiple files, and dependencies between steps," Claude Code is the right tool. If the task is "a point-to-point change at a single location," direct editing or a completion tool is faster.

---

### 5. Five-Level Evolution Model Preview

Most people, when they first encounter Claude Code, treat it as nothing more than a smarter chat window. The goal of this course is to systematically guide you through the following five levels:

| Level | Name | Your Relationship with Claude Code | Corresponding Module |
|-------|------|------------------------------------|---------------------|
| Level 1 | Conversational | You enter prompts, Claude responds and executes | Module 1 |
| Level 2 | Configuration-driven | CLAUDE.md guides behavior, memory persists across conversations | Module 2 |
| Level 3 | Automated | Hooks enforce rules, Skills encode workflows | Module 3 |
| Level 4 | Orchestrated | Custom Agents handle their own responsibilities, collaborating to complete complex tasks | Module 4 |
| Level 5 | Autonomous | Agent teams operate independently and cross-verify results | Module 5 |

The barrier to entry for Level 1 is very low, which is also why most people stop here -- it is already useful enough. But at Level 1, Claude Code has no memory, no rules, and no reusable workflows; every conversation starts from scratch.

Level 3 and above is where the real engineering value of Claude Code lies: what you build is not just a one-off assistant, but a repeatable, maintainable, and extensible automation system.

---

## Hands-on Example: Observing the Agent Loop Execute a Complete Task

> Main project continuation: As the first step in the AI Dev Assistant project, confirm that you can successfully trigger an Agent Loop.

**Goal**: Have Claude Code autonomously create a TypeScript function, and observe the Agent Loop execution process firsthand.

**Prerequisites**: Claude Code is installed and authenticated (if not yet installed, read 01-2 first).

**Steps**:

1. Open a terminal (Terminal on macOS, Git Bash or WSL on Windows)
2. Create an empty working directory:
   ```bash
   mkdir demo && cd demo
   ```
3. Start Claude Code:
   ```bash
   claude
   ```
4. Enter the following instruction at the Claude Code prompt:
   ```
   Create a TypeScript file add.ts with an add function that takes two numbers and returns their sum
   ```
5. Observe Claude's behavior. You will see it:
   - Decide which file needs to be created
   - Generate the code and write it to disk
   - Report the completed result
6. After Claude finishes, run the following in another terminal window:
   ```bash
   cat add.ts
   ```
   Confirm that the file exists and its content is correct.

**Expected Result**:

- `add.ts` automatically appears in the `demo/` directory
- The file contains an `add` function that takes two `number` parameters and returns a `number`
- Throughout the entire process, you only entered one natural language sentence -- you did not manually create a file or paste any code

**Common Errors**:

- `command not found: claude`: Claude Code is not installed, or the installation path is not in PATH. Refer to the installation section in 01-2.
- `You need to authenticate`: You have not logged in to your Anthropic account. Run `claude` and follow the terminal prompts to complete the OAuth authentication flow.
- Claude asks for permission to create a file and waits for confirmation: Under default settings, Claude requests permission before certain operations. Enter `y` to confirm.

Note: This example intentionally omits tests, git commit, and any verification steps. Its sole purpose is to let you see firsthand the core loop of "natural language input -> Agent autonomous execution -> result appears in the file system." The complete project construction workflow, including tests and version control, is introduced starting from 01-2.

---

## Key Takeaways

- Claude Code is an **agentic coding CLI**; its unit of work is a "task," not a "response"
- The Agent Loop (Read -> Decide -> Execute -> Verify) is its fundamental difference from completion-based AI tools
- The 200K context window allows it to understand the entire codebase, not just the snippets you paste in
- Suited for multi-step, multi-file tasks that require execution and verification; not suited for real-time completion or small single-point edits
- Most people stop at Level 1 (Conversational); the goal of this course is to take you to Level 3 and above

---

## Self-Assessment

After completing this chapter, make sure you can answer the following questions:

1. If someone says "Claude Code is just ChatGPT plus the ability to run code," can you point out where this description is inaccurate and explain the correct working model using the Agent Loop?

2. Your colleague says they want to use Claude Code for "real-time completion while typing." How would you respond? What alternative tool would you recommend, and why?

3. In the hands-on example you just executed, what steps did Claude Code perform? Which stage of the Agent Loop does each step correspond to?

---

📖 [Index](../00-index.md) ｜ ➡️ [Next: Installation & First Success](01-2-installation.md)

🌐 [繁體中文版](../../zh/01-認識Claude-Code/01-1-什麼是Claude-Code.md)
