# 04-3 Parallel Development & Worktree

> Use Git Worktree to run multiple Claude sessions simultaneously, converting wait time into development productivity.

## Learning Objectives

- Understand why the Claude Code Agent Loop requires a parallelization strategy
- Master the principles of Git Worktree and how the `claude --worktree` command works
- Be able to set up multiple independent sessions to develop different features concurrently
- Understand the use cases for the Writer/Reviewer pattern and Subagent isolation

## Content

### 1. Why Parallel Development Is Needed

When the Claude Code Agent Loop executes a task, it continuously reads files, writes code, and runs tests until the task is complete. This process is sequential: you give an instruction, wait for the result, review the result, and give the next instruction. For a moderately complex feature, the round-trip time can be 3 to 5 minutes, sometimes longer.

If you only have one terminal session open, those 3 to 5 minutes are pure waiting time. Multiply that by 20 tasks per day, and you have nearly 1 to 2 hours of idle time.

Engineer Boris Cherny adopted a strategy of running 5 local sessions plus 5 to 10 remote sessions simultaneously, using system notifications to coordinate when to switch attention. Within 30 days, he completed 259 Pull Requests. The key to this number is not speed, but parallelism: while each session is waiting, the other 9 sessions are working.

The prerequisite for parallel development is that each session's workspace must be isolated from the others. This is exactly why Git Worktree exists.

### 2. How Git Worktree Works

Worktree is a native Git feature that requires no additional installation. It allows you to create multiple working directories under the same repository, each with its own independent branch and file state, while sharing the same `.git` history and object database.

In one sentence: you can checkout multiple different branches from the same repo simultaneously, each operating independently in its own directory.

The `claude --worktree feature-name` command does the following:

1. Creates a new directory outside the repo root, named after the feature you specified
2. Creates a new branch with the same name and checks it out into that directory
3. Launches a full Claude Code session in this isolated working directory

This design means each Claude session can freely modify files, create new files, and execute git commits without affecting the main branch or work in other worktrees.

### 3. Parallel Development Workflow

A typical parallel development setup looks like this:

```
Terminal 1: claude --worktree feature-auth      → Develop login feature
Terminal 2: claude --worktree feature-dashboard → Develop dashboard
Terminal 3: claude -c                           → Day-to-day work on main branch
```

Three sessions run simultaneously, fully isolated from each other. Even if session 1 and session 2 both modify `src/utils.ts`, they are each working on different branches and will not overwrite each other.

The rhythm for switching attention is as follows: after giving an instruction in session 1, immediately switch to session 2 to review the previous task's results or give a new instruction. When both sessions are running, you can use session 3 to handle code reviews or documentation work. When a session finishes, a system notification lets you know it is time to check the results.

The essence of this workflow is replacing "waiting for Claude to respond" time with "working on another session" time.

### 4. Writer/Reviewer Pattern

Worktree also supports a special collaboration pattern where two Claude sessions play different roles:

| Session A (Writer) | Session B (Reviewer) |
|--------------------|----------------------|
| Implements the feature in a worktree | Reviews the implementation from the main branch or another worktree |
| Modifies code based on Reviewer feedback | Verifies whether fixes are complete and identifies edge cases |
| Submits a PR or merge request when done | Gives final approval |

The Writer session focuses on implementation without being distracted by quality assurance. The Reviewer session does not need to understand implementation details — it only needs to read the existing code and raise questions. This division of labor makes the output of both sessions more focused and higher quality.

In practice, you can explicitly state in the Reviewer session's prompt: "This is auth.ts from the feature-auth worktree. Please find all unhandled error scenarios and missing edge cases," letting Claude play a pure review role.

### 5. Subagent Worktree Isolation

When you use the Agent system in Claude Code (orchestrator calling subagents), subagents execute in the same working directory by default. If multiple subagents operate on the same set of files simultaneously, they will overwrite each other's results.

The solution is to add the `isolation: worktree` setting in the Agent frontmatter. With this setting, the orchestrator automatically creates an independent worktree for each subagent when calling it. After the subagent finishes its work, the orchestrator collects the results and merges them.

This mechanism allows you to safely have multiple subagents modify code in parallel without needing to manually coordinate write order.

## Hands-On Example: Parallel Development of Auth and Dashboard in AI Dev Assistant

> Continuing the main project: in the AI Dev Assistant project, we use two worktree sessions to develop the login authentication and dashboard feature modules simultaneously.

**Goal**: Create two independent worktree sessions to develop `auth.ts` and `dashboard.ts` in parallel, then merge both back into the main branch.

**Prerequisites**: You have completed the content from 04-2; the `ai-dev-assistant` project has git initialized with at least one commit (`git log` shows history).

**Steps**:

1. Confirm the git status in the `ai-dev-assistant` directory:
   ```bash
   git status
   git log --oneline -3
   ```
   Confirm you are on the `main` branch and the working directory is clean (no changes to commit).

2. Open the first terminal and start the auth worktree session:
   ```bash
   claude --worktree feature-auth
   ```
   Claude will automatically create the `feature-auth` branch and launch a session in the new directory.

3. Give the following instruction in session 1:
   ```
   Create auth.ts in the src/ directory. Implement a validateToken function.
   The function takes a string token and returns { valid: boolean; userId?: string }.
   If the token is empty or shorter than 10 characters, return { valid: false }.
   ```

4. Do not wait for session 1 to finish. Immediately open the second terminal and start the dashboard worktree session:
   ```bash
   claude --worktree feature-dashboard
   ```

5. Give the following instruction in session 2:
   ```
   Create dashboard.ts in the src/ directory. Implement a getMetrics function.
   The function returns { activeUsers: number; requestsPerMinute: number; errorRate: number }.
   Initialize all values to 0, and add a JSDoc comment explaining the meaning of each field.
   ```

6. Both sessions are now working in parallel. You can switch between the two terminals to observe their progress, or give the next instruction as soon as one session finishes while the other continues running.

7. After both sessions are complete, return to the main branch to merge:
   ```bash
   git checkout main
   git merge feature-auth
   git merge feature-dashboard
   ```

8. Confirm that both files exist on the main branch:
   ```bash
   ls src/
   # Expected to see auth.ts and dashboard.ts
   ```

9. Clean up completed worktrees (Claude automatically cleans up when a session ends, but you can also do it manually):
   ```bash
   git worktree list
   git worktree remove feature-auth
   git worktree remove feature-dashboard
   ```

**Expected Results**:
- The `feature-auth` and `feature-dashboard` branches each have their own independent commit history
- File modifications from the two sessions during development are fully isolated and do not affect each other
- After merging into `main`, both `src/auth.ts` and `src/dashboard.ts` exist with complete functionality

**Common Errors**:

- **`fatal: 'feature-auth' is already checked out at '/path/to/worktree'`**: The same branch name can only be checked out in one worktree at a time. If you previously created a worktree with the same name but did not clean it up, this error will occur. Solution: run `git worktree list` to see existing worktrees, use `git worktree remove` to remove the old one, or use a different branch name.

- **Merge conflicts (`CONFLICT (content): Merge conflict in src/utils.ts`)**: If both sessions modified the same shared file (e.g., `utils.ts` or `index.ts`), conflicts will occur during the merge. This is identical to a standard git merge conflict — use `git status` to identify conflicting files, manually edit to resolve them, then run `git add` and `git commit`. To prevent this, clearly define each session's file scope before starting, and avoid having two sessions touch the same file.

## Key Takeaways

- Git Worktree is a native Git feature that lets you checkout multiple branches from the same repo into different directories simultaneously, each with fully independent file state
- `claude --worktree feature-name` automatically creates a worktree and launches an isolated Claude session inside it, with no manual git operations required
- The core value of parallel development is converting Agent Loop wait time into productive work time in other sessions, rather than relying purely on speed
- The Writer/Reviewer pattern gives two sessions clear division of labor, each focusing on either implementation or review, avoiding the blind spots that occur when a single session both writes and reviews
- The same branch name cannot exist in two worktrees simultaneously; each session must use a unique branch name

## Self-Assessment

- Can you start a second session and begin working without closing the first Claude session? Try it and confirm that `git branch` in each worktree shows a different branch name.
- If both sessions modify `src/config.ts`, what happens during the merge? Can you use `git diff feature-auth feature-dashboard -- src/config.ts` to preview the differences before merging?
- In Boris Cherny's strategy of completing 259 PRs with 5 local sessions, where is the bottleneck? Is it Claude's speed, your speed at switching attention, or the frequency of git merges?

---

⬅️ [Previous: Context Management Strategies](04-2-context-management.md) ｜ 📖 [Index](../00-index.md) ｜ ➡️ [Next: Agent Teams](../05-autonomous/05-1-agent-teams.md)

🌐 [繁體中文版](../../zh/04-編排層/04-3-平行開發與Worktree.md)
