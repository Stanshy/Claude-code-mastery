# 03-3 Skills 系統
> 將不常用但關鍵的工作流程封裝成可按需呼叫的 Skill，讓 context 只在真正需要時才被消耗。

## 學習目標
- 說明 Skills 與 CLAUDE.md 在載入時機上的根本差異
- 掌握 SKILL.md frontmatter 的必填欄位與作用
- 使用 `$ARGUMENTS` 與 `` !`command` `` 注入動態內容
- 建立並成功呼叫一個具備四維分析能力的 `/review` Skill

---

## 正文

### 1. 為什麼需要 Skills

CLAUDE.md 是一個永遠載入的配置檔——每次會話開始，不論你正在做什麼，它的全部內容都會佔用 context window。這個設計適合「每次對話都有用的規則」：程式碼風格、提交格式、語氣要求。

但不是所有知識都需要一直存在。考慮這些場景：

- **部署流程**：你每兩週部署一次，但流程有 15 個步驟和環境差異說明
- **Release checklist**：每季執行一次，需要詳細的品質門檻核對清單
- **Prisma schema 規範**：只有在修改資料模型時才需要，平時無用

如果把這些放進 CLAUDE.md，代價是：每次開啟 Claude Code——即使你只是寫一個 utility function——都要消耗這些 context。Skills 解決的就是這個問題：**只在明確需要時才載入**，不用則不消耗。

David Chu 的畢業路徑對此有清楚的定位：Skills 適用於模型未經訓練的知識（如你公司的內部工具使用方式）、不常使用的工作流程（如部署）、以及你希望以特定方式完成的任務（如結構化 code review）。

---

### 2. Skill vs CLAUDE.md 差異

理解差異是選擇正確工具的前提。兩者互補，不互斥：

| 屬性 | CLAUDE.md | Skill |
|------|-----------|-------|
| 載入時機 | 每次會話自動 | 按需（手動或 Claude 判斷） |
| 大小限制 | 建議 < 200 行 | 無限制 |
| 呼叫方式 | 不可呼叫 | `/skill-name` |
| 適用 | 永遠適用的規則 | 特定工作流或知識 |

決策依據：問自己「這個知識在每次對話都有用嗎？」答案是否，就屬於 Skill。

---

### 3. 建立 Skill

Skill 是一個含 YAML frontmatter 的 Markdown 檔案，放在固定目錄下。

**位置**

- 專案級別：`.claude/skills/<skill-name>/SKILL.md`（隨版本控制，團隊共用）
- 使用者級別：`~/.claude/skills/<skill-name>/SKILL.md`（跨所有專案有效）

建議優先使用專案級別，讓 Skill 與程式碼一起進版本控制。

**Frontmatter 關鍵欄位**

每個 SKILL.md 開頭必須包含 YAML frontmatter：

| 欄位 | 說明 | 預設值 |
|------|------|--------|
| `name` | Skill 的呼叫名稱，對應 `/name` 指令 | 必填 |
| `description` | 讓 Claude 判斷何時自動使用此 Skill | 必填 |
| `disable-model-invocation` | 設 `true` 則只能手動呼叫，不會自動觸發 | `false` |
| `allowed-tools` | 限制此 Skill 可使用的工具清單 | 所有工具 |
| `context` | 設 `fork` 則在隔離上下文中執行 | 無 |

`description` 的品質直接影響 Skill 能否被 Claude 自動發現。好的 description 描述「何時適合使用」，而不是「這個 Skill 做什麼」。

---

### 4. 動態內容注入

靜態 Skill 只能執行固定任務。兩種注入機制讓 Skill 感知真實狀態：

**`$ARGUMENTS`：接收使用者傳入的參數**

```
Review the following files: $ARGUMENTS
```

使用者輸入 `/review src/index.ts` 時，`$ARGUMENTS` 的值為 `src/index.ts`。

若有多個參數，可用 `$0`、`$1`、`$2` 按位置取得：

```
/deploy staging v1.2.0
```

此時 `$0` = `staging`，`$1` = `v1.2.0`。

**`` !`command` ``：執行 bash 並注入輸出**

```
Current staged diff:
!`git diff --staged`
```

Skill 載入時，這段語法會立即執行 `git diff --staged`，並將輸出結果直接插入 prompt。Claude 在開始分析之前就看到了真實資料，而不是空的佔位符。

兩種機制可以組合使用：

```markdown
Diff for: $ARGUMENTS
!`git diff $ARGUMENTS`
```

---

### 5. 什麼時候不該用 Skill

Skills 是工具箱的其中一格，選錯工具會導致問題：

- 規則需要每次都生效 → 用 **CLAUDE.md**（如程式碼風格、語言偏好）
- 規則需要 100% 強制執行、不允許 Claude 跳過 → 用 **Hook**（如 03-1 的 pre-commit 驗證）
- 角色需要跨多種任務、能自主分解子任務 → 用 **Agent**（04-1 章節）

---

## 實作範例：為 AI Dev Assistant 建立 /review Skill
> 主案例延續：在 AI Dev Assistant 專案中，手動 code review 沒有統一標準，不同人關注的面向不同。我們建立一個 `/review` Skill，讓 Claude 每次都從 Functionality、Security、Performance、Maintainability 四個維度產出結構化報告。

**目標**：建立 `/review` Skill，自動讀取 staged git diff 並產出含 severity 評級的四維分析報告

**前置條件**：
- 已完成 03-1（Hooks 基礎），`.claude/` 目錄已存在
- 在 `ai-dev-assistant` 專案根目錄操作
- 專案已初始化 git（`git init`）

**步驟**：

**步驟 1**：建立 Skill 目錄

```bash
mkdir -p .claude/skills/review
```

**步驟 2**：建立 `.claude/skills/review/SKILL.md`

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

**步驟 3**：確認目錄結構

```
ai-dev-assistant/
└── .claude/
    └── skills/
        └── review/
            └── SKILL.md
```

**步驟 4**：建立一個含有已知問題的測試檔案，模擬真實開發變更

```bash
cat > src/auth/validateToken.ts << 'EOF'
export function validateToken(token: string): boolean {
  // 直接字串比對，有 timing attack 風險
  return token === process.env.SECRET_TOKEN;
}
EOF
```

**步驟 5**：將變更加入 staging area

```bash
git add src/auth/validateToken.ts
```

執行後用 `git diff --staged --stat` 確認有輸出，這是 Skill 能讀到資料的前提。

**步驟 6**：啟動 Claude Code

```bash
claude
```

**步驟 7**：輸入 `/review` 並觀察執行順序

Claude 的執行流程：
1. 載入 `.claude/skills/review/SKILL.md`
2. 執行 `` !`git diff --staged --stat` ``，取得變更摘要
3. 執行 `` !`git diff --staged` ``，取得完整 diff
4. 將真實資料填入 prompt，開始四維分析

**步驟 8**：閱讀產出的結構化報告

報告應包含類似以下格式：

```
## Review Report: src/auth/validateToken.ts

**File**: src/auth/validateToken.ts
**Line**: 3
**Severity**: critical
**Description**: 直接使用 === 比對 token，易受 timing attack 攻擊
**Suggestion**: 改用 crypto.timingSafeEqual() 進行常數時間比對

**File**: src/auth/validateToken.ts
**Line**: 3
**Severity**: warning
**Description**: 若 SECRET_TOKEN 未設定，函式永遠回傳 false，且無任何錯誤提示
**Suggestion**: 在程式啟動時驗證必要環境變數，使用 zod 或 envalid
```

**步驟 9**：驗證 `disable-model-invocation: true` 的作用

在 Claude Code 中輸入「幫我 review 目前的程式碼」（不使用 `/review` 指令）。由於設定了 `disable-model-invocation: true`，Claude 不應自動觸發 Skill，只會用一般對話方式回應。這是有副作用操作的安全機制。

**步驟 10**：將 Skill 加入版本控制

```bash
git add .claude/skills/review/SKILL.md
git commit -m "feat: add structured code review skill"
```

Skill 進入版本控制後，整個團隊執行 `/review` 都會使用相同的標準。

**預期結果**：
- `/review` 在 Claude Code 中可用，並產出含四個維度的結構化報告
- 每個 finding 包含檔案路徑、行號、severity 評級與具體修改建議
- 未使用 `/review` 明確呼叫時，Claude 不會自動觸發此 Skill

**常見錯誤**：

**錯誤 1：輸入 `/review` 後無回應**

原因：SKILL.md 的 frontmatter YAML 語法錯誤。YAML 對格式極為敏感，任何缺少空格或多餘縮排都會導致解析失敗。

診斷方式：
- 確認檔案以 `---` 開頭，frontmatter 結尾也有 `---`
- 確認 `name:` 後有一個空格再接值
- 確認沒有使用 Tab 縮排（YAML 只接受空格）

正確格式：
```yaml
---
name: review
description: Perform a structured code review on staged or specified changes
disable-model-invocation: true
---
```

**錯誤 2：報告顯示「No changes found」或 diff 為空**

原因：`` !`git diff --staged` `` 只讀取已 `git add` 的 staged changes。如果沒有 staged changes，輸出就是空的，Claude 沒有任何內容可以分析。

解決方式：執行 `/review` 前先執行 `git add <files>`，再用 `git diff --staged --stat` 確認有輸出後再呼叫 Skill。

---

## 關鍵重點
- Skills 的核心價值是「按需載入」——只有在明確呼叫時才消耗 context，是 CLAUDE.md 的互補工具，而不是替代品
- `description` 欄位決定 Claude 能否自動發現 Skill；應描述「何時適合使用」，而不是「這個 Skill 做什麼」
- `` !`command` `` 語法在 Skill 載入時注入真實的 shell 輸出，讓 Claude 在分析前就掌握最新狀態
- 有副作用的操作（部署、通知、資料庫寫入）必須設定 `disable-model-invocation: true`，防止 Claude 在錯誤時機自動觸發
- Skills 進入版本控制後，整個團隊使用相同的工作流程標準，避免個人差異

## 自我檢核
- 你的 CLAUDE.md 中有哪些內容只在特定任務（如部署、release）時才有用？這些應該遷移到獨立的 Skill。
- 如果 `/deploy` Skill 沒有設定 `disable-model-invocation: true`，當使用者說「幫我把這個推上去」時可能發生什麼？
- `` !`git diff --staged` `` 與 `` !`git diff` `` 的輸出有何不同？在什麼情境下該用哪一個？

---

⬅️ [上一章：Hooks 進階與 Validator 模式](03-2-Hooks進階與Validator模式.md) ｜ 📖 [回目錄](../00-索引與摘要.md) ｜ ➡️ [下一章：MCP Servers](03-4-MCP-Servers.md)

🌐 [English Version](../../en/03-automation/03-3-skills.md)
