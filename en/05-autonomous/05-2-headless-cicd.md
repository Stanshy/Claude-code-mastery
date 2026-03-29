# 05-2 Headless Mode & CI/CD Integration
> Let Claude Code automatically execute tasks in unattended environments, automating everything from PR reviews to documentation updates.

## Learning Objectives
- Understand how Headless mode (`-p` flag) works and the differences between the three output formats
- Use `--session-id` to maintain context across multiple non-interactive invocations
- Set up a GitHub Actions workflow that automatically triggers a Claude Code review on every PR
- Identify common permission and configuration issues in CI/CD integrations

## Content

### 1. Why Headless Mode Is Needed

Up to this point, every Claude Code operation has required you to sit at the terminal, type commands, confirm actions, and read output. This interactive model is intuitive during development, but it has a fundamental limitation: when you are not there, it does nothing.

Yet there is an entire class of tasks whose ideal execution time is precisely "when you are not there":

- A colleague opens a PR while you are asleep, but the PR needs a code review
- A feature merges into main, and CI should automatically update the API documentation
- A security vulnerability scan runs at 3 AM every night, with a report ready by morning

If these tasks wait for manual execution, they are either delayed or forgotten. Headless mode (`-p` flag) solves this problem: it lets Claude Code accept a prompt, execute it to completion, and then exit -- the entire process requires no human interaction. This allows Claude Code to be invoked by any automation system -- shell scripts, GitHub Actions, cron jobs, and more.

### 2. `-p` Non-Interactive Mode

`-p` is short for `--print`, meaning "execute and print the result, then exit."

```bash
# Most basic usage: ask a question and get an answer
claude -p "Explain the architecture of this project"

# Output structured results in JSON format for programmatic parsing
claude -p "List all API endpoints" --output-format json

# Streaming JSON: output incrementally, suitable for scenarios requiring real-time progress
claude -p "Analyze code quality in src/" --output-format stream-json
```

Criteria for choosing among the three output formats:

| Format | Purpose | Suitable Scenarios |
|--------|---------|-------------------|
| text (default) | Plain text, human-readable | Results displayed to users, written to log files |
| json | Single JSON object, output all at once after task completion | Programs need to parse results, write to databases |
| stream-json | Each chunk is a JSON object, output incrementally during execution | Long-running tasks requiring real-time progress display |

In shell scripts, the `json` format lets you use `jq` to extract fields precisely:

```bash
# Extract the issues array from Claude's analysis results
claude -p "Analyze security issues in src/auth.ts, return as JSON" \
  --output-format json | jq '.issues[]'
```

### 3. Multi-Turn Conversations: Maintaining Context with `--session-id`

By default, `-p` mode is stateless: each invocation starts a brand-new session with no memory. However, some automation workflows require an "analyze first, then act on the analysis" pattern, which needs context to persist across invocations.

The `--session-id` parameter lets multiple `-p` invocations share the same session:

```bash
# Step 1: Analyze the problem
claude -p "Analyze bugs in src/auth.ts, list all issues" \
  --session-id fix-auth-session

# Step 2: Act within the same session (Claude remembers the analysis from step 1)
claude -p "Fix all the issues you just analyzed" \
  --session-id fix-auth-session

# Step 3: Verify the results
claude -p "Confirm the fixed src/auth.ts has no remaining issues" \
  --session-id fix-auth-session
```

The session ID is an arbitrary string; you are responsible for the naming convention. A recommended approach is to use the task name plus a date or PR number, such as `fix-auth-2026-03-24` or `review-pr-142`, so that session records in logs remain readable.

### 4. GitHub Actions Integration

GitHub Actions is the standard CI/CD tool and the most common deployment environment for Headless Claude Code. Anthropic provides the official `claude-code-action`, which directly wraps authentication, invocation, and output into a single step.

**Installation Method 1 (Recommended)**: Run the slash command in an interactive Claude Code session, and it will guide you through the GitHub App installation and permission setup:

```
/install-github-app
```

**Installation Method 2 (Manual)**: Create the workflow file directly in the project for full control over configuration details (see the hands-on example below).

`claude-code-action` supports two trigger modes:
- **Automatic trigger**: Automatically executes on GitHub events such as PR opening, pushes to specific branches, etc.
- **Manual trigger (`@claude`)**: Mention @claude in a PR or issue comment, and Claude will respond and execute the corresponding task

### 5. Automation Scenarios and Workflow Design

Below are four practical automation scenarios showing how to combine GitHub events with Claude Code tasks:

**Automatic PR Code Review**:
```
Trigger: pull_request.opened / pull_request.synchronize
Task: Analyze the diff, leave specific comments on functionality correctness, security issues, and performance concerns
Output: Post review comments directly on the PR
```

**Auto-Update Documentation After Merge**:
```
Trigger: push to main
Task: Compare changes in src/, update the corresponding API description sections in the README
Output: Commit the updated README to main
```

**Scheduled Security Scan**:
```
Trigger: schedule (cron expression)
Task: Scan dependencies in package.json, identify known CVEs
Output: Open a security advisory ticket in GitHub Issues
```

**Auto-Fix Lint Failures**:
```
Trigger: workflow_run (after the lint workflow fails)
Task: Read lint errors, fix all auto-fixable rule violations
Output: Commit the fixed files to the same branch
```

When designing automation workflows, two principles are worth remembering. First, make Claude's task description as specific as possible -- "review this PR" produces far worse results than "analyze the changes and provide specific line-level feedback on functionality, security, and performance." Second, Claude operates on a real repo in the CI environment, so configure the minimum required permissions -- do not grant `contents: write` unless the task genuinely needs write access.

## Hands-On Example: Automatic Code Review with GitHub Actions
> Main project continuation: In the AI Dev Assistant project, set up automatic code review execution every time a PR is opened, having Claude leave specific comments on the PR.

**Goal**: Set up a GitHub Actions workflow so that every time someone opens or updates a PR for ai-dev-assistant, Claude automatically analyzes the diff and leaves structured review comments on the PR.

**Prerequisites**:
- Completed 03-3 (/review Skill setup)
- ai-dev-assistant is pushed to GitHub with a corresponding remote repo
- You have admin or secrets management permissions for the repo
- You hold a valid Anthropic API key

**Steps**:

1. Try the official installation command in an interactive Claude Code session (skip steps 2-3 if successful):
   ```
   /install-github-app
   ```
   Follow the terminal prompts to complete the GitHub App authorization flow. If you encounter network restrictions or need finer-grained control, continue to step 2.

2. Manually create the workflow directory and file:
   ```bash
   mkdir -p /c/projects/ai-dev-assistant/.github/workflows
   ```

3. Create `.github/workflows/claude-review.yml` with the following content:
   ```yaml
   name: Claude Code Review
   on:
     pull_request:
       types: [opened, synchronize]

   jobs:
     review:
       runs-on: ubuntu-latest
       permissions:
         contents: read
         pull-requests: write
       steps:
         - uses: actions/checkout@v4
           with:
             fetch-depth: 0
         - uses: anthropics/claude-code-action@v1
           with:
             anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
             prompt: |
               Review this PR for the following aspects:
               1. Functionality correctness - does the logic achieve what the PR describes?
               2. Security issues - any hardcoded secrets, injection risks, or unsafe operations?
               3. Performance concerns - unnecessary loops, missing indexes, or expensive operations?
               4. Code quality - TypeScript type safety, naming clarity, ESLint rule compliance
               Provide specific line-level comments for each issue found.
               For each comment, include the severity (critical / warning / suggestion).
   ```

4. Set the API key secret in the GitHub repo: Go to the repo page, then Settings, then Secrets and variables, then Actions, then New repository secret. Enter `ANTHROPIC_API_KEY` as the name and your API key as the value.

5. Push the workflow file to the repo:
   ```bash
   cd /c/projects/ai-dev-assistant
   git add .github/workflows/claude-review.yml
   git commit -m "ci: add Claude Code automatic PR review workflow"
   git push origin main
   ```

6. Create a test PR. Branch off main, make an arbitrary change to a TypeScript file, push it, and open a PR:
   ```bash
   git checkout -b test/trigger-claude-review
   echo "// test comment" >> src/index.ts
   git add src/index.ts
   git commit -m "test: trigger Claude review workflow"
   git push origin test/trigger-claude-review
   ```
   Then open a PR for this branch on GitHub.

7. Observe the workflow execution status in the Actions tab of the GitHub repo. On success, Claude's review comments will appear in the PR's Files Changed tab after approximately 30-60 seconds.

8. Verify that the comment format meets expectations: each comment should include the issue type, a specific description, and a severity label (critical / warning / suggestion).

**Expected Results**:
- After the PR is opened, GitHub Actions automatically triggers the `Claude Code Review` workflow
- The workflow shows a green (success) status on the Actions page
- Claude's review comments appear in the PR's Files Changed tab, each with a clear severity and specific remediation suggestion
- If the PR has no issues, Claude also leaves a "No critical issues found" confirmation comment

**Common Errors**:
- **`Error: ANTHROPIC_API_KEY is not set` causes workflow failure**: This error is visible in the workflow log on the Actions page. Solution: confirm that you added a secret with the exact correct name under Settings, then Secrets and variables, then Actions (the name is case-sensitive). After setting the secret, you need to re-trigger the workflow (e.g., push a new commit to the PR branch).
- **Workflow succeeds but no comments appear on the PR**: The most common cause is the workflow lacking `pull-requests: write` permission. Check the `permissions` block and confirm `pull-requests: write` is present. Another possibility is that `actions/checkout` is missing `fetch-depth: 0`, preventing Claude from obtaining the full diff.

## Key Takeaways
- The `-p` flag makes Claude Code execute once and exit, forming the foundation for all automation integrations
- `--session-id` lets multiple `-p` invocations share context, enabling two-step "analyze then act" automation
- `anthropics/claude-code-action@v1` wraps authentication and invocation for CI environments, providing the shortest path to GitHub Actions integration
- The minimum permissions Claude needs in GitHub Actions are `contents: read` plus `pull-requests: write` -- do not over-grant
- The more specific the prompt, the more consistent the review quality in CI; "review this PR" is the worst possible prompt -- listing specific check items yields actionable results

## Self-Assessment
- Can you explain the difference in use cases between `--output-format json` and `--output-format stream-json`, and give a specific scenario suited to each?
- If a Claude review workflow in GitHub Actions executes successfully but the PR has no comments, what three things would you check, and in what order?
- Using `--session-id`, can you design a three-step automation script (analyze, fix, verify)?
