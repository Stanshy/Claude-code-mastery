# 05-3 Agent SDK
> Control Claude Code programmatically and embed AI capabilities into your own applications and automation tools.

## Learning Objectives
- Understand the fundamental differences between the Agent SDK and the CLI, and the criteria for choosing between them
- Be able to install and use the TypeScript SDK's `query` API to execute tasks
- Master the roles of two core parameters: `allowedTools` and `maxTurns`
- Build an `assistant` CLI command for ai-dev-assistant that can be executed from any directory

## Content

### 1. Why You Need the Agent SDK

The CLI is designed for developers sitting directly in front of a terminal. Its interactive interface, real-time output, and confirmation prompts all assume "someone is watching the screen." When your needs shift from "I want to use Claude Code" to "I want my program to call Claude Code," the CLI is no longer the right tool.

The Agent SDK provides a programmatic interface that turns Claude Code's capabilities into a function call within your code. This unlocks several scenarios that the CLI cannot handle:

**Embedding in Existing Applications**: Your Node.js backend automatically calls Claude Code when a user submits a bug report, analyzes the relevant code, generates fix suggestions, and returns them directly to the user — no human intervention required.

**Custom CLI Tools**: Your team has specific workflows that need a single command to chain together "analyze PR," "generate changelog," and "update documentation" — all done in one go.

**CI Pipeline Integration**: Within a CI script, dynamically determine which fix strategy Claude Code should execute based on the type of test failure, and pass the results as a structured object to the next pipeline step.

The SDK is available in both Python and TypeScript. This chapter focuses on TypeScript, since ai-dev-assistant is itself a TypeScript project and can be directly integrated.

### 2. SDK vs CLI

Understanding the differences between the two helps you choose the right tool for the right scenario:

| Aspect | CLI | SDK |
|--------|-----|-----|
| Usage | Terminal interaction, manual input | Code invocation, automated execution |
| Target Users | Developers in daily work | Tool developers, automation engineers |
| Output Format | Formatted text, designed for human readability | Structured objects, designed for programmatic processing |
| Session Management | Manual operation (`--session-id`) | Programmatic control, decisions based on logic |
| Error Handling | Displayed in terminal, manual judgment | try/catch, programmatic handling |
| Integration Difficulty | Requires shell calls or child_process | Direct import, native integration |

The key decision question: "Am I going to execute this task myself, or do I want a program to execute it for me?" Use the CLI for the former, the SDK for the latter.

### 3. Installation

The SDK package names correspond to each language:

```bash
# TypeScript / Node.js projects
npm install @anthropic-ai/claude-agent-sdk

# Python projects
pip install claude-agent-sdk
```

Make sure the environment variable is set before installation — the SDK uses the same API key:

```bash
echo $ANTHROPIC_API_KEY  # Must have a value, otherwise SDK calls will throw an error
```

### 4. Core API

The main entry point of the TypeScript SDK is the `query` function. It accepts a prompt and options, and returns an async iterator that you can consume with `for await...of` to process Claude's responses incrementally:

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Find and fix the bug in src/fixer.ts",
  options: {
    allowedTools: ["Read", "Edit", "Bash", "Grep", "Glob"],
    maxTurns: 20,
  },
})) {
  if (message.type === "text") {
    console.log(message.content);
  }
}
```

The equivalent Python version:

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    async for message in query(
        prompt="Find and fix the bug in src/fixer.ts",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Edit", "Bash", "Grep", "Glob"],
            max_turns=20,
        ),
    ):
        if message.type == "text":
            print(message.content)

asyncio.run(main())
```

The two most important options parameters:

**`allowedTools`**: Explicitly lists the tools Claude is allowed to use. If a tool is not listed, it is disabled. This is the first line of defense for security boundaries. For tasks that only need to read and write code, `["Read", "Edit", "Bash", "Grep", "Glob"]` is sufficient. If the task involves web queries, add `"WebSearch"` and `"WebFetch"`.

**`maxTurns`**: The maximum number of tool-call rounds Claude can execute in a single `query` invocation. One round represents a single "Claude thinks → calls a tool → gets the result" cycle. Complex tasks (analyzing multiple files, multi-step fixes) require more turns. Start with 20, observe actual consumption, then adjust. Setting it too low will cause the task to be truncated midway.

### 5. Built-in Tools

The SDK provides the exact same tool set as the CLI, referenced by string name in `allowedTools`:

| Tool Name | Function | Common Use Cases |
|-----------|----------|------------------|
| `Read` | Read file contents | Analyze code |
| `Write` | Create new files | Generate new modules |
| `Edit` | Modify existing files | Fix bugs, refactor |
| `Bash` | Execute shell commands | Run tests, build |
| `Glob` | File pattern matching | Find files of a specific type |
| `Grep` | Content search | Find all usages of a function |
| `WebSearch` | Search the web | Look up latest docs or CVEs |
| `WebFetch` | Fetch web page content | Retrieve documentation at a specific URL |
| `AskUserQuestion` | Ask the user | Decision points that require human confirmation |

Principle: Only grant the tools the task actually needs. The narrower `allowedTools` is, the more predictable the behavior, and the lower the probability of unexpected side effects.

## Hands-on Example: Building an AI Dev Assistant CLI Entry Point with the SDK
> Main project continuation: In the AI Dev Assistant project, use the TypeScript SDK to build an `assistant` CLI command so that users can run `assistant "Fix the login bug"` from any directory to invoke Claude Code to execute the task.

**Goal**: Build a globally available `assistant` CLI tool for ai-dev-assistant. The user inputs a task description, Claude autonomously completes it and outputs the result in the terminal — no need to enter the Claude Code interactive interface.

**Prerequisites**:
- Completed 04-1 (Subagents setup), ai-dev-assistant project structure is complete
- Node.js 18.x, TypeScript 5.x, `ts-node` or `tsc` available
- `ANTHROPIC_API_KEY` environment variable is set
- `package.json` already has a `build` script (`tsc`)

**Steps**:

1. Install the Agent SDK:
   ```bash
   cd /c/projects/ai-dev-assistant
   npm install @anthropic-ai/claude-agent-sdk
   ```
   After installation, verify that the `node_modules/@anthropic-ai/claude-agent-sdk` directory exists.

2. Create the CLI entry point logic in `src/index.ts` (if the file already exists, replace with the following content; if it doesn't exist, create it):
   ```typescript
   #!/usr/bin/env node
   import { query } from "@anthropic-ai/claude-agent-sdk";

   const task = process.argv.slice(2).join(" ");

   if (!task) {
     console.error("Usage: assistant <task>");
     console.error('Example: assistant "Fix the login bug in src/auth.ts"');
     process.exit(1);
   }

   async function main(): Promise<void> {
     console.log(`Running: ${task}\n`);

     for await (const message of query({
       prompt: task,
       options: {
         allowedTools: ["Read", "Edit", "Bash", "Grep", "Glob"],
         maxTurns: 30,
       },
     })) {
       if (message.type === "text") {
         process.stdout.write(message.content);
       }
     }

     console.log("\nDone.");
   }

   main().catch((err) => {
     console.error("Error:", err.message);
     process.exit(1);
   });
   ```

3. Add the `bin` field in `package.json` so npm knows this package provides an executable command:
   ```json
   {
     "bin": {
       "assistant": "./dist/index.js"
     }
   }
   ```
   Note: `bin` points to the compiled `dist/index.js`, not `src/index.ts`.

4. Verify that `tsconfig.json` has `outDir` set to `"./dist"` and `target` set to `ES2020` or higher (to support `for await...of`):
   ```json
   {
     "compilerOptions": {
       "target": "ES2020",
       "module": "commonjs",
       "outDir": "./dist",
       "strict": true
     }
   }
   ```

5. Compile TypeScript:
   ```bash
   npm run build
   ```
   After successful compilation, `dist/index.js` should exist.

6. Add execute permissions to the compiled file (required on Linux/macOS; can be skipped on Windows but does not affect `npm link`):
   ```bash
   chmod +x dist/index.js
   ```

7. Use `npm link` to link this package globally so the `assistant` command is available anywhere on the system:
   ```bash
   npm link
   ```
   `npm link` creates a symbolic link in the global node_modules pointing to the current project directory.

8. Test the command from any directory to verify it works:
   ```bash
   cd /tmp
   assistant "List all TypeScript files under the src/ directory of ai-dev-assistant and explain the purpose of each file"
   ```
   Claude will use the Glob tool to find the files, then use the Read tool to read each file, and finally output the explanations.

**Expected Results**:
- `assistant "any task description"` can be executed from any directory on the system
- The terminal displays Claude's thought process and execution results in real time (streamed output)
- After the task completes, `Done.` is displayed and the process exits normally (exit code 0)
- If the task description is empty, a Usage prompt is displayed and the process exits with exit code 1

**Common Errors**:
- **`Cannot find module '@anthropic-ai/claude-agent-sdk'` error**: The SDK is not installed, or `npm install` was run in the wrong directory. Solution: Make sure you run `npm install @anthropic-ai/claude-agent-sdk` in the `/c/projects/ai-dev-assistant` directory, and verify that the package name appears in the `dependencies` section of `package.json`.
- **`Permission denied: dist/index.js` when running `assistant` (Linux/macOS)**: `dist/index.js` is missing the executable bit, and the first line is missing the shebang. Solution: Ensure the first line of `src/index.ts` is `#!/usr/bin/env node` (note this must exist before compilation), re-run `npm run build`, then run `chmod +x dist/index.js`.
- **`assistant` command not found after `npm link` (command not found)**: The npm global bin directory is not in PATH. Run `npm bin -g` to get the global bin directory path, add that path to the `PATH` variable in `~/.bashrc` or `~/.zshrc`, reload the shell, and try again.

## Key Takeaways
- The core value of the Agent SDK is "letting code call Claude Code," not replacing the CLI's interactive use cases
- `query` returns an async iterator; consuming streamed output with `for await...of` is the most natural approach
- `allowedTools` is the security boundary — only grant the tools the task actually needs; overly broad permissions increase the risk of unexpected behavior
- `maxTurns` controls task depth; setting it too low causes complex tasks to be truncated — start with 20, observe, then adjust
- `npm link` is the standard approach for local CLI tool development; after publishing to npm, users switch to `npm install -g`

## Self-Assessment
- Can you explain what types of tasks `allowedTools: ["Read", "Grep", "Glob"]` and `allowedTools: ["Read", "Edit", "Bash", "Grep", "Glob"]` are each suited for, and why the distinction matters?
- If `assistant "Fix the bug in src/auth.ts"` stops outputting halfway through and displays a truncation message, which parameter would you adjust, and what is a reasonable starting value?
- From the SDK's perspective, what practical problem does the async iterator design of the `query` function solve (as opposed to returning the complete result all at once)?

---

⬅️ [Previous: Headless Mode & CI/CD Integration](05-2-headless-cicd.md) ｜ 📖 [Index](../00-index.md) ｜ ➡️ [Next: Seven Design Principles](../06-harness/06-1-seven-principles.md)

🌐 [繁體中文版](../../zh/05-自主層/05-3-Agent-SDK.md)
