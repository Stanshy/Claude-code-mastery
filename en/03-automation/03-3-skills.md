# 03-3 Skills System
> Encapsulate infrequently used but critical workflows into on-demand Skills, so context is only consumed when truly needed.

## Learning Objectives
- Explain the fundamental difference between Skills and CLAUDE.md in terms of loading timing
- Master the required fields and their purposes in SKILL.md frontmatter
- Use `$ARGUMENTS` and `` !`command` `` to inject dynamic content
- Build and successfully invoke a `/review` Skill with four-dimensional analysis capability

---

## Content

### 1. Why Skills Are Needed

CLAUDE.md is a configuration file that is always loaded—every time a session starts, regardless of what you are doing, its entire content occupies the context window. This design is suitable for "rules that are useful in every conversation": code style, commit format, tone requirements.

But not all knowledge needs to be present at all times. Consider these scenarios:

- **Deployment process**: You deploy every two weeks, but the process has 15 steps and environment-specific instructions
- **Release checklist**: Executed quarterly, requiring a detailed quality gate checklist
- **Prisma schema conventions**: Only needed when modifying data models, useless otherwise

If you put these in CLAUDE.md, the cost is: every time you open Claude Code—even if you are just writing a utility function—these consume context. Skills solve exactly this problem: **loaded only when explicitly needed**, consuming nothing when unused.

David Chu's graduation path provides a clear positioning for this: Skills are suited for knowledge the model has not been trained on (such as how to use your company's internal tools), infrequently used workflows (such as deployment), and tasks you want completed in a specific way (such as structured code review).

---

### 2. Skill vs CLAUDE.md Differences

Understanding the differences is a prerequisite for choosing the right tool. The two are complementary, not mutually exclusive:

| Attribute | CLAUDE.md | Skill |
|-----------|-----------|-------|
| Loading timing | Automatic every session | On-demand (manual or Claude-determined) |
| Size limit | Recommended < 200 lines | Unlimited |
| Invocation method | Cannot be invoked | `/skill-name` |
| Suited for | Rules that always apply | Specific workflows or knowledge |

Decision criterion: Ask yourself "Is this knowledge useful in every conversation?" If the answer is no, it belongs in a Skill.

---

### 3. Creating a Skill

A Skill is a Markdown file with YAML frontmatter, placed in a fixed directory.

**Location**

- Project-level: `.claude/skills/<skill-name>/SKILL.md` (version-controlled, shared across the team)
- User-level: `~/.claude/skills/<skill-name>/SKILL.md` (effective across all projects)

It is recommended to use project-level first, so Skills are version-controlled alongside the code.

**Frontmatter Key Fields**

Each SKILL.md must begin with YAML frontmatter:

| Field | Description | Default |
|-------|-------------|---------|
| `name` | The invocation name of the Skill, corresponding to the `/name` command | Required |
| `description` | Helps Claude determine when to automatically use this Skill | Required |
| `disable-model-invocation` | Set to `true` to allow only manual invocation, preventing automatic triggering | `false` |
| `allowed-tools` | Restricts the list of tools this Skill can use | All tools |
| `context` | Set to `fork` to execute in an isolated context | None |

The quality of `description` directly affects whether Claude can automatically discover the Skill. A good description describes "when it is appropriate to use," not "what this Skill does."

---

### 4. Dynamic Content Injection

A static Skill can only perform fixed tasks. Two injection mechanisms allow Skills to be aware of real-time state:

**`$ARGUMENTS`: Receive parameters passed by the user**

```
Review the following files: $ARGUMENTS
```

When the user inputs `/review src/index.ts`, the value of `$ARGUMENTS` is `src/index.ts`.

For multiple arguments, use `$0`, `$1`, `$2` to access by position:

```
/deploy staging v1.2.0
```

Here `$0` = `staging`, `$1` = `v1.2.0`.

**`` !`command` ``: Execute bash and inject the output**

```
Current staged diff:
!`git diff --staged`
```

When the Skill is loaded, this syntax immediately executes `git diff --staged` and inserts the output directly into the prompt. Claude sees real data before it starts analyzing, rather than an empty placeholder.

The two mechanisms can be combined:

```markdown
Diff for: $ARGUMENTS
!`git diff $ARGUMENTS`
```

---

### 5. When Not to Use Skills

Skills are one compartment in the toolbox—choosing the wrong tool leads to problems:

- Rules need to take effect every time → Use **CLAUDE.md** (e.g., code style, language preferences)
- Rules need 100% enforcement with no possibility of Claude skipping them → Use **Hooks** (e.g., pre-commit validation from 03-1)
- Roles need to work across multiple tasks and autonomously decompose subtasks → Use **Agents** (Chapter 04-1)

---

## Hands-On Example: Building a /review Skill for AI Dev Assistant
> Main case continuation: In the AI Dev Assistant project, manual code reviews lack a unified standard, and different people focus on different aspects. We build a `/review` Skill so that Claude always produces a structured report from four dimensions: Functionality, Security, Performance, and Maintainability.

**Goal**: Build a `/review` Skill that automatically reads staged git diff and produces a four-dimensional analysis report with severity ratings

**Prerequisites**:
- Completed 03-1 (Hooks basics), `.claude/` directory already exists
- Operating in the `ai-dev-assistant` project root directory
- Project has git initialized (`git init`)

**Steps**:

**Step 1**: Create the Skill directory

```bash
mkdir -p .claude/skills/review
```

**Step 2**: Create `.claude/skills/review/SKILL.md`

```markdown
---
name: review
description: Perform a structured code review on staged or specified changes
disable-model-invocation: true
allowed-tools: Read, Grep, Glob, Bash
---

# Code Review

Review the following changes: $ARGUMENTS

If no arguments provided, review staged changes.

## Context
!`git diff --staged --stat`

## Full Diff
!`git diff --staged`

## Review Checklist
Analyze from 4 angles:

### 1. Functionality
- Does the code do what it claims?
- Are edge cases handled?

### 2. Security
- Any hardcoded secrets?
- Input validation present?

### 3. Performance
- Unnecessary loops or allocations?
- Database query efficiency?

### 4. Maintainability
- Clear naming?
- Appropriate abstraction level?

## Output Format
For each finding:
- **File**: path
- **Line**: number
- **Severity**: critical / warning / suggestion
- **Description**: what and why
- **Suggestion**: how to fix
```

**Step 3**: Verify the directory structure

```
ai-dev-assistant/
└── .claude/
    └── skills/
        └── review/
            └── SKILL.md
```

**Step 4**: Create a test file with known issues to simulate a real development change

```bash
cat > src/auth/validateToken.ts << 'EOF'
export function validateToken(token: string): boolean {
  // Direct string comparison, vulnerable to timing attacks
  return token === process.env.SECRET_TOKEN;
}
EOF
```

**Step 5**: Add the changes to the staging area

```bash
git add src/auth/validateToken.ts
```

After executing, use `git diff --staged --stat` to confirm there is output—this is a prerequisite for the Skill to read data.

**Step 6**: Launch Claude Code

```bash
claude
```

**Step 7**: Enter `/review` and observe the execution sequence

Claude's execution flow:
1. Load `.claude/skills/review/SKILL.md`
2. Execute `` !`git diff --staged --stat` `` to get a change summary
3. Execute `` !`git diff --staged` `` to get the full diff
4. Fill real data into the prompt and begin four-dimensional analysis

**Step 8**: Read the generated structured report

The report should contain a format similar to:

```
## Review Report: src/auth/validateToken.ts

**File**: src/auth/validateToken.ts
**Line**: 3
**Severity**: critical
**Description**: Using === for token comparison is vulnerable to timing attacks
**Suggestion**: Use crypto.timingSafeEqual() for constant-time comparison

**File**: src/auth/validateToken.ts
**Line**: 3
**Severity**: warning
**Description**: If SECRET_TOKEN is not set, the function always returns false with no error indication
**Suggestion**: Validate required environment variables at application startup using zod or envalid
```

**Step 9**: Verify the effect of `disable-model-invocation: true`

In Claude Code, type "help me review the current code" (without using the `/review` command). Since `disable-model-invocation: true` is set, Claude should not automatically trigger the Skill and will only respond in a regular conversational manner. This is a safety mechanism for operations with side effects.

**Step 10**: Add the Skill to version control

```bash
git add .claude/skills/review/SKILL.md
git commit -m "feat: add structured code review skill"
```

Once the Skill is in version control, the entire team uses the same standard when running `/review`.

**Expected Results**:
- `/review` is available in Claude Code and produces a structured report with four dimensions
- Each finding includes file path, line number, severity rating, and specific fix suggestions
- When `/review` is not explicitly invoked, Claude does not automatically trigger this Skill

**Common Errors**:

**Error 1: No response after entering `/review`**

Cause: YAML syntax error in SKILL.md frontmatter. YAML is extremely sensitive to formatting—any missing space or extra indentation will cause parsing to fail.

Diagnosis:
- Confirm the file starts with `---` and the frontmatter also ends with `---`
- Confirm there is a space after `name:` before the value
- Confirm no Tab indentation is used (YAML only accepts spaces)

Correct format:
```yaml
---
name: review
description: Perform a structured code review on staged or specified changes
disable-model-invocation: true
---
```

**Error 2: Report shows "No changes found" or diff is empty**

Cause: `` !`git diff --staged` `` only reads staged changes that have been `git add`-ed. If there are no staged changes, the output is empty and Claude has nothing to analyze.

Solution: Before running `/review`, first run `git add <files>`, then use `git diff --staged --stat` to confirm there is output before invoking the Skill.

---

## Key Takeaways
- The core value of Skills is "on-demand loading"—context is consumed only when explicitly invoked. Skills are a complementary tool to CLAUDE.md, not a replacement
- The `description` field determines whether Claude can automatically discover a Skill; it should describe "when it is appropriate to use," not "what this Skill does"
- The `` !`command` `` syntax injects real shell output when the Skill is loaded, giving Claude the latest state before analysis begins
- Operations with side effects (deployment, notifications, database writes) must set `disable-model-invocation: true` to prevent Claude from automatically triggering at the wrong time
- Once Skills are in version control, the entire team uses the same workflow standards, avoiding individual differences

## Self-Assessment
- Which content in your CLAUDE.md is only useful during specific tasks (such as deployment or release)? These should be migrated to separate Skills.
- If a `/deploy` Skill does not have `disable-model-invocation: true` set, what might happen when a user says "help me push this up"?
- What is the difference in output between `` !`git diff --staged` `` and `` !`git diff` ``? In what scenarios should you use which one?
