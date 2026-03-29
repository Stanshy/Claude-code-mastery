# 03-4 MCP Servers
> MCP (Model Context Protocol) lets Claude Code break beyond the local file system boundary, interacting directly with external systems like GitHub, databases, and Slack — without manually writing integration scripts.

## Learning Objectives
- Explain the fundamental problem MCP solves and how it differs from manual API calls
- Master both CLI and `.mcp.json` installation methods and their respective use cases
- Understand the priority order of the three scopes (project, user, session)
- Successfully connect to GitHub MCP and verify that Claude can directly query repo data

---

## Content

### 1. Why MCP Is Needed

By default, Claude Code can only operate on two types of resources: the local file system and shell commands. This design is sufficient in most cases, but real-world development requires interacting with external systems:

- Read GitHub Issues to understand requirement context
- Query PostgreSQL to analyze data distribution
- Send deployment notifications via Slack
- Take browser screenshots to verify UI changes

Without MCP, the traditional approach is: manually run `curl` → copy the response → paste into Claude Code → Claude analyzes → repeat the whole process when follow-up queries are needed. The fundamental problem with this workflow is that **the data is static** — Claude cannot autonomously initiate subsequent queries.

MCP (Model Context Protocol) is an open standard that defines the communication protocol between Claude and external tool servers. After installing an MCP server, Claude can directly call external APIs just like calling local Bash tools — including autonomously deciding when to query, what content to query, and continuing to advance the task based on results.

Analogy: Hooks are Claude Code's "automatic triggers," Skills are "work manuals," and MCP is the "external toolbox."

---

### 2. How MCP Works

An MCP Server is an independently running program that exposes a set of named tools for Claude Code to use. Claude Code communicates with it in two ways:

| Type | Communication Method | Use Case |
|------|---------------------|----------|
| `stdio` | stdin / stdout | Local programs (e.g., servers launched via npx) |
| `http` | HTTP / HTTPS | Remote APIs (e.g., GitHub's official MCP endpoint) |

Once a connection is established, Claude calls MCP tools when it needs external data. The MCP server executes the actual API requests and returns the results to Claude. The entire process is transparent to the user — you see Claude directly providing results rather than asking you to manually copy and paste.

---

### 3. Installation Methods

MCP servers have two installation methods, corresponding to different use cases.

**Method 1: CLI Command**

```bash
claude mcp add github https://api.github.com/mcp/ --scope user
```

The CLI method is fast and suitable for quickly setting up well-known official MCP servers. `--scope user` makes the configuration apply to all projects; omitting it defaults to project-level (local scope).

**Method 2: Configuration File `.mcp.json`**

Create `.mcp.json` in the project root directory:

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.github.com/mcp/",
      "headers": {
        "Authorization": "Bearer $GITHUB_TOKEN"
      }
    }
  }
}
```

The advantage of the configuration file approach is that it can be version-controlled, allowing the entire team to use the same MCP settings. Note that sensitive information like `$GITHUB_TOKEN` must be injected via environment variables — **never write them directly in the configuration file**.

---

### 4. Scopes

The scope of an MCP server determines in which contexts it takes effect:

| Location | Range | Use Case |
|----------|-------|----------|
| Project `.mcp.json` | Current project only | Project-specific tools, settings that need to be shared across the team |
| `~/.claude/.mcp.json` | All projects for the current user | Frequently used personal tools (GitHub, Slack) |
| `--mcp-config` CLI parameter | Current session only | Temporarily testing a new MCP server |

When multiple scopes define an MCP server with the same name, **the project-level `.mcp.json` has the highest priority** and can override user-level settings. This allows specific projects to use different versions or different endpoints of the same service.

---

### 5. Tool Search Auto Mode

Enabling multiple MCP servers has an easily overlooked context cost issue.

Each MCP server needs to describe all its available tools to Claude when connecting. When you have 5 MCP servers enabled simultaneously, each with over 10 tools, the tool descriptions alone can consume approximately 55,000 tokens. This overhead occurs at the start of every session, even if that particular task only uses one of the tools.

Tool Search uses a "discover on demand" mechanism: instead of loading all tool descriptions at the start of a session, Claude searches for relevant tools when needed. This mechanism can reduce context consumption by approximately 85% (to about 8,700 tokens).

Recommendations for managing multiple MCP servers:
- Only enable MCP servers that are actually needed for the current task
- For infrequently used servers, prefer using `--mcp-config` to enable them on a per-session basis
- Periodically run `/mcp` to confirm which servers are in a connected state

---

### 6. Recommended MCP Servers

| Server | Purpose | Installation Command |
|--------|---------|---------------------|
| GitHub | Repo / PR / issue operations | `claude mcp add github https://api.github.com/mcp/` |
| Playwright | Browser automation / screenshots | `claude mcp add playwright --type stdio npx @anthropic-ai/mcp-playwright` |
| PostgreSQL | Database queries | `claude mcp add postgres --type stdio npx @anthropic-ai/mcp-postgres` |

---

### 7. When Not to Use MCP

MCP is not a universal solution. The following scenarios are better handled directly with the shell:

**One-off simple API calls**: Fetching a file's content from GitHub once is faster with a direct `curl` command — no need to install and configure an entire MCP server.

**High-security scenarios**: MCP servers operate external systems on your behalf, and the risk is especially high when they have write permissions. For third-party MCP servers not officially from Anthropic, you should review their source code before installation.

MCP's value lies in tasks requiring **multi-step, dynamic interaction** — where Claude needs to decide what to query next based on the results of the first query. One-off operations do not justify introducing this additional layer of complexity.

---

## Hands-on Example: Connecting GitHub MCP to AI Dev Assistant
> Continuing the main case study: In the AI Dev Assistant project, developers frequently need to check GitHub Issues to understand feature requirements, currently by manually copying API responses and pasting them into Claude. We'll connect GitHub MCP so Claude can directly read Issues and search PRs, eliminating this manual bridging step.

**Goal**: Set up the GitHub MCP Server so Claude can directly read Issues, create PRs, and verify that multi-step queries chain together automatically

**Prerequisites**:
- Completed 02-2 (settings.json configuration)
- Have a GitHub account
- Have created a GitHub Personal Access Token (requires `repo` and `read:org` permissions)

**Steps**:

**Step 1**: Obtain a GitHub Personal Access Token

Go to GitHub Settings → Developer settings → Personal access tokens → Fine-grained tokens, create a new token, select the repositories you need access to, and ensure permissions include at least `Contents: Read` and `Issues: Read and Write`.

**Step 2**: Set the environment variable

```bash
export GITHUB_TOKEN=ghp_xxxxx
```

It is recommended to add this line to `~/.bashrc` or `~/.zshrc` to make it permanent. A more persistent approach is to add it to the `env` field in `.claude/settings.local.json`:

```json
{
  "env": {
    "GITHUB_TOKEN": "ghp_xxxxx"
  }
}
```

Note: `settings.local.json` should not be version-controlled. Make sure `.gitignore` includes this file.

**Step 3**: Create `.mcp.json` in the project root directory

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.github.com/mcp/",
      "headers": {
        "Authorization": "Bearer $GITHUB_TOKEN"
      }
    }
  }
}
```

**Step 4**: Confirm the `GITHUB_TOKEN` environment variable is set

```bash
echo $GITHUB_TOKEN
```

It should display a token starting with `ghp_` or `github_pat_`. If it is empty, go back to Step 2.

**Step 5**: Launch Claude Code

```bash
claude
```

**Step 6**: Use `/mcp` to confirm the GitHub server is connected

```
/mcp
```

You should see:

```
MCP Servers:
  github    connected
```

If it shows `disconnected` or `error`, resolve the authentication issue before continuing (see Common Errors).

**Step 7**: Execute your first MCP query

Enter:

```
List the 5 most recently updated repos in my GitHub account
```

Observe Claude's behavior: it will show that it is calling a GitHub MCP tool, then directly return a structured list of repos including names, last update times, and languages. The entire process requires no manual `curl` execution or copy-pasting.

**Step 8**: Test a multi-step query (verifying MCP's core value)

Enter:

```
Look at the 3 most recent open issues in the ai-dev-assistant repo and tell me which has the highest priority
```

Observe how Claude: first queries the issue list, reads the detailed content of each issue, judges priority based on labels and descriptions, and finally provides a recommendation. This query requires multiple MCP calls, and the parameters of each call depend on the results of the previous one — this is precisely the scenario where MCP has a fundamental advantage over manual API calls.

**Expected Results**:
- `/mcp` shows the github server status as connected
- Claude can read your GitHub repo list without manually running `curl`
- Multi-step queries (e.g., "find the highest priority issue") can automatically chain multiple API calls and consolidate results

**Common Errors**:

**Error 1: `401 Unauthorized` or `/mcp` shows disconnected**

Cause: The token is invalid, expired, or the `$GITHUB_TOKEN` environment variable is not set correctly.

Resolution steps:
1. Run `echo $GITHUB_TOKEN` to confirm it has a value
2. In GitHub Settings, confirm the token has not expired and the scope includes `repo`
3. If the token is correct but it still fails, verify the `.mcp.json` JSON format is valid (use `cat .mcp.json | python3 -m json.tool` to verify)
4. Restart Claude Code

**Error 2: `/mcp` shows connected but queries return no response or empty results**

Cause: The token has insufficient permissions, or the Fine-grained token does not have the correct repository scope selected.

Resolution steps:
1. Confirm the token's `Repository access` includes the target repo (or select All repositories)
2. Confirm the token permissions include `Contents: Read` and `Issues: Read and Write`
3. If using a Fine-grained token, confirm the token's expiration has not passed
4. Switch to a Classic token with `repo` scope selected as a testing baseline

---

## Key Takeaways
- The core problem MCP solves is transforming Claude from passively receiving data to actively querying data, eliminating the need to manually bridge external systems
- Version-controlling `.mcp.json` lets teams share MCP configurations, but sensitive tokens must be injected via environment variables — never hardcode them in configuration files
- The context consumption of multiple simultaneously enabled MCP servers should not be ignored (5 servers ≈ 55,000 tokens); only enable servers needed for the current task, and leverage Tool Search auto mode
- For one-off query scenarios, using `curl` directly is simpler; MCP's value lies in tasks requiring multi-step, dynamic interaction
- Only install MCP servers from trusted sources; third-party servers operate external systems on your behalf — review their source code before installation

## Self-Assessment
- In your current development workflow, which steps require manually calling external APIs and pasting results into Claude? These are potential use cases for MCP.
- If you have GitHub, Slack, PostgreSQL, Figma, and Playwright — five MCP servers enabled simultaneously — but a particular task only requires querying the database, how should you manage context consumption?
- In a high-security production environment, what criteria would you use to evaluate whether a third-party MCP server is trustworthy?
