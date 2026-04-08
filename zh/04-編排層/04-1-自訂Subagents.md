# 04-1 自訂 Subagents
> 將複雜角色從主對話中獨立出來，讓 Claude Code 以乾淨的 context 執行專屬任務。

## 學習目標
- 理解 Subagent 的核心價值：獨立 context，精簡結果回傳主對話
- 區分內建 Subagent 與自訂 Subagent 的適用場景
- 掌握 `.claude/agents/<name>.md` 的 frontmatter 關鍵欄位與撰寫要點
- 選擇正確的調用方式（自動委派、@-mention、Session-wide）

---

## 正文

### 1. 為什麼需要 Subagents

主對話的 context 是有限資源，而且是最容易被忽視的資源。

當你請 Claude Code 調查一個複雜問題時，過程中會產生大量中間產物：讀取了哪些檔案、工具調用的輸出、嘗試過但失敗的方向、逐步縮小範圍的推理鏈。這些中間產物對調查過程有價值，但對主對話幾乎毫無意義——你真正需要的只是最終答案。

Subagent 的核心價值正是切斷這條污染路徑：Subagent 在自己的獨立 context 中執行完整的調查過程，只把精簡的結果回傳給主對話。主對話的 context 不受影響，可以繼續處理後續任務。

**David Chu 的畢業路徑**說明了角色定義的演進：初學者從 `CLAUDE.md` 中寫角色描述開始，熟悉後進階到 Skills（可重複的工作流），最終建立自訂 Agent（擁有人設的持久角色）。兩者的分野在此：Skill 適合「單一可重複的工作流」（例如部署、格式化），Agent 適合「跨多種任務都需要同一個角色」（例如安全審查員、除錯專家）。為 Skill 定義任務，為 Agent 定義人設。

---

### 2. 內建 vs 自訂 Subagent

Claude Code 內建了三種 Subagent，各有明確的使用邊界：

| Subagent | 模型 | 工具 | 主要用途 |
|---|---|---|---|
| Explore | claude-haiku | 只讀（Read, Grep, Glob） | 快速搜尋 codebase，成本低 |
| Plan | 繼承主對話 | 只讀 | 規劃模式中的需求研究 |
| general-purpose | 繼承主對話 | 所有工具 | 複雜多步驟任務 |
| 自訂 | 可指定 | 可指定 | 擁有特定人設的專屬角色 |

內建 Subagent 的設計是通用的。當你的專案需要一個「每次審查 auth 模組都用同一套標準的安全審查員」，或是「專門負責隔離並修復 bug 的除錯專家」，內建 Subagent 無法承載這樣的人設與限制——這就是自訂 Subagent 的出發點。

---

### 3. 建立自訂 Subagent：檔案結構與欄位說明

自訂 Subagent 是一個放在 `.claude/agents/` 目錄下的 Markdown 檔案。檔案名稱即為 Agent 的識別名稱（小寫，使用連字符）。

**標準檔案位置**

```
.claude/
└── agents/
    ├── security-reviewer.md
    └── debugger.md
```

**Frontmatter 關鍵欄位**

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

各欄位的作用與注意事項：

- **`name`**：識別名稱，用於 @-mention 調用。必須與檔案名稱一致，使用小寫連字符格式。
- **`description`**：這是最重要的欄位。Claude Code 依據 description 決定是否自動委派任務給這個 Agent。描述必須清楚寫明「觸發條件」，例如「Use after any changes to authentication」，而非模糊的「reviews security」。
- **`tools`**：允許的工具列表。精確限制工具是安全性的基礎——安全審查員只需要讀取權限，不應該有 Edit 或 Bash。
- **`model`**：指定要使用的模型。高精確度任務（安全審查、架構決策）用 `claude-sonnet-4-5` 或 `claude-opus-4-5`；高頻低複雜度任務用 `claude-haiku-4-5` 節省成本。若省略，繼承主對話的模型。
- **`memory: project`**：啟用持久記憶，Agent 可以跨 session 記住專案特定的知識。
- **`isolation: worktree`**：讓 Agent 在獨立的 git worktree 中執行，避免修改影響主工作目錄（適合 debugger 這類需要做實驗的 Agent）。

**Frontmatter 之後的 System Prompt**

Frontmatter 下方的 Markdown 內容就是這個 Agent 的 system prompt。這裡定義角色的思維框架、審查標準、輸出格式。撰寫時的原則：具體而非抽象。「Review code for SQL injection, XSS, command injection」比「Review code for security issues」更能讓 Agent 執行一致的工作。

---

### 4. 三種調用方式

自訂 Subagent 建立後，有三種方式可以調用：

**自動委派（推薦日常使用）**

當你對主對話輸入一個任務，Claude Code 會掃描所有 Agent 的 description，判斷是否應該自動委派。例如你說「幫我確認這個 PR 的 auth 模組沒有安全問題」，Claude Code 看到 security-reviewer 的 description 包含「authentication」，會自動委派。這是最不需要額外操作的方式，但依賴 description 的品質。

**@-mention（需要明確指定時）**

```
@security-reviewer 檢查 src/auth/ 目錄
@debugger 這個測試失敗了，幫我找出原因
```

當你確定要用某個 Agent，或自動委派沒有觸發時使用。

**Session-wide（整個 session 都使用同一個 Agent）**

```bash
claude --agent security-reviewer
```

從命令列啟動時指定，適合整個 session 都在做同一類型的任務（例如一整個下午專門做安全審查）。

---

### 5. Skill vs Agent：選擇判斷表

| 需求 | 用 Skill | 用 Agent |
|---|---|---|
| 單一固定流程（如部署、格式化） | ✓ | |
| 跨多種任務的持久角色（如 reviewer） | | ✓ |
| 需要獨立 context 避免污染主對話 | | ✓ |
| 需要指定特定模型（如 haiku 降低成本） | | ✓ |
| 需要精確限制工具權限 | | ✓ |
| 需要 git worktree 隔離實驗環境 | | ✓ |
| 一次性的快速任務 | ✓ | |

Skill 的本質是「把一段固定的 prompt 模板化」；Agent 的本質是「賦予一個持久角色特定的工具、模型與行為框架」。如果你發現自己每次都要重複寫同樣的角色描述，那就是建立 Agent 的時機。

---

## 實作範例：建立 security-reviewer 和 debugger Agent
> 主案例延續：在 AI Dev Assistant 專案中，建立兩個專用 Agent，分別負責安全審查與除錯隔離。

**目標**：為 ai-dev-assistant 專案建立 security-reviewer 和 debugger 兩個 Agent，並驗證調用行為與工具限制是否正確運作。

**前置條件**：已完成 03-3（Skills），ai-dev-assistant 專案目錄結構存在，包含 `src/` 目錄。

**步驟**：

**步驟 1**：建立 agents 目錄

```bash
mkdir -p .claude/agents
```

**步驟 2**：建立 `.claude/agents/security-reviewer.md`

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

**步驟 3**：建立 `.claude/agents/debugger.md`

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

**步驟 4**：啟動 Claude Code

```bash
claude
```

**步驟 5**：確認 Agent 已被識別

在主對話中輸入：

```
列出所有可用的 agents
```

預期看到 security-reviewer 和 debugger 出現在清單中。

**步驟 6**：測試 security-reviewer 的 @-mention 與工具限制

```
@security-reviewer 檢查 src/ 目錄是否有安全漏洞
```

觀察：security-reviewer 只使用 Read、Grep、Glob，不嘗試執行任何修改操作。主對話只收到審查報告，不會看到 Agent 讀取每個檔案的過程。

**步驟 7**：測試 debugger 的 @-mention

先在 `src/reviewer.ts` 中製造一個已知的問題，或使用現有的失敗測試，然後輸入：

```
@debugger 調查 src/reviewer.test.ts 的測試失敗
```

觀察：debugger 的調查過程（讀取檔案、執行測試、追蹤 call stack）在獨立 context 中進行，主對話只收到「根因、修復內容、測試通過」的精簡摘要。

**步驟 8**：驗證自動委派

不使用 @-mention，直接輸入：

```
幫我確認剛才加入的 auth token 驗證邏輯沒有安全問題
```

觀察 Claude Code 是否自動選擇 security-reviewer，而不是自己處理。如果沒有自動選擇，回頭檢查 description 中的觸發關鍵字是否足夠明確。

**預期結果**：
- 兩個 Agent 均可透過 @-mention 正確觸發
- security-reviewer 的工具限制生效：只使用讀取類工具，無法執行 Bash 或修改檔案
- debugger 可以讀取、修改檔案並執行 `npm test`
- 主對話 context 不會被 Agent 的調查過程膨脹，始終保持精簡

**常見錯誤**：

**錯誤 1：Agent 不回應 @-mention，或自動委派不觸發**

原因：description 欄位太模糊，Claude Code 無法判斷觸發時機。

錯誤範例：`description: Reviews security issues`

正確做法：在 description 中明確寫出觸發條件與關鍵字，例如：`description: Reviews code for security vulnerabilities. Use after any changes to authentication, authorization, API key handling, or data handling code. Trigger when asked to check security, review auth, or audit access control.`

**錯誤 2：Agent 嘗試使用未在 tools 欄位列出的工具，導致操作被拒絕**

原因：撰寫 system prompt 時要求 Agent 執行的操作，超過了 tools 欄位授權的工具範圍。例如 security-reviewer 的 system prompt 要求執行 `npm audit`，但 tools 欄位沒有包含 Bash。

正確做法：在撰寫 system prompt 之前，先確定 tools 欄位的清單，system prompt 中只描述這些工具能完成的操作。

---

## 關鍵重點
- Subagent 的根本價值在於「獨立 context」：複雜的調查過程留在 Subagent，主對話只拿到精簡結果
- `description` 欄位決定自動委派是否觸發，必須包含明確的觸發條件與關鍵字，而非模糊的功能描述
- `tools` 欄位是工具的白名單，不列出的工具一律無法使用，這是最小權限原則的具體實踐
- Skill 對應「可重複的工作流」，Agent 對應「需要持久角色與獨立 context 的任務」，兩者不互斥
- `isolation: worktree` 讓需要做實驗或修改檔案的 Agent 在獨立的 git 環境中運行，保護主工作目錄

## 自我檢核
- 如果 @security-reviewer 沒有被自動委派觸發，你的第一個排查步驟是什麼？
- security-reviewer 的 `tools` 欄位設定為 `Read, Grep, Glob`，但你希望它也能執行 `npm audit`，應該如何調整，同時維持最小權限原則？
- 在什麼情況下你會選擇建立 Agent 而非 Skill？請舉出一個具體的 ai-dev-assistant 專案情境。

---

⬅️ [上一章：MCP Servers](../03-自動化層/03-4-MCP-Servers.md) ｜ 📖 [回目錄](../00-索引與摘要.md) ｜ ➡️ [下一章：上下文管理策略](04-2-上下文管理策略.md)

🌐 [English Version](../../en/04-orchestration/04-1-subagents.md)
