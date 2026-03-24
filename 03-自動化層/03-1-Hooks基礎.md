# 03-1 Hooks 基礎
> Hooks 是「保證」，不是「建議」——學會它，讓 Claude Code 的行為變成系統級強制執行。

## 學習目標
- 理解 Hooks 與 CLAUDE.md 規則在可靠度上的根本差異
- 識別 6 個最實用的 Hook 事件及其觸發時機
- 掌握 `settings.json` 的 hooks 配置格式與欄位意義
- 為 ai-dev-assistant 設定 3 個具體的基礎 Hook 並驗證運作

---

## 正文

### 1. 為什麼需要 Hooks

你在 CLAUDE.md 寫了「每次修改 TypeScript 檔案後必須跑 ESLint」。第一週，Claude 確實照做。第三週，它開始「遺忘」——特別是在長對話或壓縮後的 session 裡。

這不是 bug，這是設計。CLAUDE.md 裡的規則本質上是「建議」——模型將它們納入上下文，自行決定是否執行。當上下文視窗飽和、或模型判斷規則與當前任務衝突時，建議就會被跳過。

Hooks 解決的是這個可靠度問題。Hooks 是系統層級的攔截器——在工具執行前後，由 Claude Code 的執行環境強制呼叫，與模型的意志無關。模型無法「決定不跑」一個 PostToolUse Hook，就像程式無法決定不讓作業系統的 signal handler 觸發一樣。

**David Chu 的畢業路徑**：當你發現 Claude 不再持續遵守 CLAUDE.md 裡的某條規則，就是它該「畢業」成 Hook 的時候。偏好和風格留在 CLAUDE.md；安全、驗證、格式化升級到 Hooks。

---

### 2. 確定性 vs. 建議性

| 屬性 | CLAUDE.md 規則 | Hook |
|------|---------------|------|
| 性質 | 建議 | 強制 |
| 執行 | 模型自覺 | 系統攔截 |
| 可靠度 | 80–95% | 100% |
| 範圍 | 當次上下文 | 所有 Session |
| 適用場景 | 偏好、風格、語氣 | 安全、驗證、格式化 |

這個表格的關鍵列是「可靠度」。80–95% 聽起來不差，但在安全場景裡，5% 的失敗率是不可接受的。如果你有一條規則「不得修改 `.env` 檔案」，你需要的是 100%，不是 95%。

---

### 3. 常用 Hook 事件

Claude Code 提供多個 Hook 事件點。以下聚焦最實用的 6 個：

| 事件 | 觸發時機 | 典型用途 |
|------|---------|---------|
| `PreToolUse` | 工具執行前（可阻擋操作） | 保護敏感檔案、驗證命令安全性 |
| `PostToolUse` | 工具執行後（操作已完成） | 自動格式化、記錄稽核日誌 |
| `Stop` | Claude 完成完整回應時 | 強制驗證（Validator Pattern） |
| `SessionStart` | 會話開始或從壓縮狀態恢復時 | 載入外部上下文、重注入核心規則 |
| `PermissionRequest` | 出現權限提示前 | 自動批准已知安全操作 |
| `UserPromptSubmit` | 使用者送出提示前 | 輸入前處理、附加固定指令 |

`PreToolUse` 是最強大的事件：它是唯一能「阻止」操作發生的 Hook。其餘事件在操作已發生後才觸發，只能做事後處理。

---

### 4. Hook 配置格式

Hook 設定在 `.claude/settings.json` 的 `hooks` 欄位中。結構如下：

```json
{
  "hooks": {
    "事件名稱": [
      {
        "matcher": "工具名稱（選填）",
        "hooks": [
          {
            "type": "command",
            "command": "要執行的 shell 命令"
          }
        ]
      }
    ]
  }
}
```

每個欄位的精確意義：

- **`事件名稱`**（最外層 key）：對應上表的事件，如 `PreToolUse`、`PostToolUse`。
- **`matcher`**：比對工具名稱的字串。支援 `|` 分隔多個工具，如 `"Edit|Write"`。省略時比對所有工具。
- **`type`**：處理器類型。本章只使用 `"command"`（執行 shell 命令）。進階類型（如 `"mcp"`）在 03-2 說明。
- **`command`**：實際執行的 shell 命令字串。命令透過 stdin 接收 JSON 格式的事件資料。

注意巢狀結構：外層陣列是「規則群組」，每個群組有自己的 matcher；`hooks` 陣列是該群組下的處理器清單。一個事件可以有多個群組，執行順序由上到下。

---

### 5. Hook 的輸入與輸出

Hook 命令透過 stdin 接收 JSON 格式的事件資料。以 `PreToolUse` 攔截 `Edit` 工具為例：

```json
{
  "session_id": "abc123",
  "hook_event_name": "PreToolUse",
  "tool_name": "Edit",
  "tool_input": {
    "file_path": "src/index.ts",
    "old_string": "const x = 1",
    "new_string": "const x = 2"
  }
}
```

這代表你的 Hook 腳本可以用 `jq` 或任何 JSON 解析工具讀取 `tool_input.file_path`，判斷是否要阻擋這次操作。

**退出碼是 Hook 與系統溝通的唯一語言：**

| 退出碼 | 意義 | 附加效果 |
|--------|------|---------|
| `exit 0` | 允許操作繼續 | stdout 內容作為上下文回傳給 Claude |
| `exit 2` | 阻擋操作（僅 PreToolUse 有效） | stderr 內容作為反饋告訴 Claude，說明為何被阻擋 |
| 其他非 0 值 | 視為 Hook 執行錯誤 | 預設行為依設定而定（通常允許繼續） |

`exit 2` 的 stderr 訊息非常重要——Claude 會讀取它，並依此調整策略。寫清楚的錯誤訊息，讓 Claude 知道「為什麼被擋」，而不只是「被擋了」。

---

## 實作範例：為 AI Dev Assistant 設定 3 個基礎 Hook

> 主案例延續：在 AI Dev Assistant 專案中，你已完成 settings.json 基礎設定（02-2）。現在要把三條關鍵規則從「建議」升級為「保證」。

**目標**：
為 ai-dev-assistant 加入 3 個 Hook：
1. `PostToolUse`：編輯 TypeScript 檔案後自動執行 ESLint 修正
2. `PreToolUse`：保護 `.env` 系列檔案不被修改
3. `SessionStart`：壓縮後重新注入核心技術規則

**前置條件**：
- 已完成 02-2（`.claude/settings.json` 已建立）
- 專案根目錄已有 `package.json` 且 ESLint 8.x 已安裝
- 系統已安裝 `jq`（用於解析 JSON stdin）

**步驟**：

**步驟 1**：建立 hooks 目錄。

```bash
mkdir -p .claude/hooks
```

**步驟 2**：建立保護敏感檔案的 Hook 腳本 `.claude/hooks/protect-files.sh`。

```bash
#!/bin/bash
# 從 stdin 讀取事件 JSON
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

# 定義受保護的檔案模式
PROTECTED=(".env" ".env.local" ".env.production" ".env.staging")

for pattern in "${PROTECTED[@]}"; do
  if [[ "$FILE_PATH" == *"$pattern"* ]]; then
    echo "Blocked: cannot modify '$FILE_PATH' — this is a protected environment file." >&2
    echo "Suggestion: if you need to update environment variables, describe the change and let the developer apply it manually." >&2
    exit 2
  fi
done

exit 0
```

**步驟 3**：設定腳本執行權限（缺少此步驟 Hook 會靜默失敗）。

```bash
chmod +x .claude/hooks/protect-files.sh
```

**步驟 4**：開啟 `.claude/settings.json`，在現有 `permissions` 設定之後加入 `hooks` 欄位。完整結構如下：

```json
{
  "permissions": {
    "allow": ["Read", "Edit", "Write", "Bash"],
    "deny": []
  },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "FILE=$(echo \"$(cat)\" | jq -r '.tool_input.file_path // empty'); if [[ \"$FILE\" == *.ts || \"$FILE\" == *.tsx ]]; then npx eslint --fix \"$FILE\" 2>/dev/null; fi; exit 0"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/protect-files.sh"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "matcher": "compact",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Context restored after compaction. Project: ai-dev-assistant. Stack: Node.js 18.x, TypeScript 5.x, Jest 29.x, ESLint 8.x. Critical rules: (1) run npm test before marking any task complete, (2) never modify .env files directly, (3) all new files must have corresponding .test.ts files.'"
          }
        ]
      }
    ]
  }
}
```

**步驟 5**：在 Claude Code 中開啟 ai-dev-assistant 專案，請 Claude 編輯任意一個 `.ts` 檔案（例如在 `src/index.ts` 中新增一行程式碼）。

觀察終端機輸出：ESLint 應在編輯完成後自動執行並修正格式問題。

**步驟 6**：請 Claude 修改 `.env` 檔案（例如「請在 .env 中新增 `LOG_LEVEL=debug`」）。

觀察 Claude 的回應：它應收到 "Blocked" 訊息，並轉換策略（如改為告知你應該手動新增哪些內容）。

**步驟 7**：執行 `/hooks` 命令確認所有 Hook 正確載入，並確認事件與 matcher 顯示正確。

**預期結果**：
- 編輯任何 `.ts` 或 `.tsx` 檔案後，ESLint `--fix` 自動執行，格式問題靜默修正
- 嘗試修改 `.env`、`.env.local` 等檔案時，Claude 收到明確的阻擋訊息並提供替代建議
- `/hooks` 輸出顯示 3 個 Hook 已啟用，分別掛載在 `PostToolUse`、`PreToolUse`、`SessionStart` 事件上
- 壓縮後重新開啟 session，Claude 能從 `SessionStart` Hook 的 stdout 中讀取技術棧提醒

**常見錯誤**：

**錯誤 1：`jq: command not found`**

現象：Hook 腳本執行時立即失敗，Claude 收不到任何有效的阻擋訊息或允許訊號。

原因：`jq` 未安裝。大多數 Linux/macOS 環境預設不含 jq。

解決方式：
- macOS：`brew install jq`
- Ubuntu/Debian：`sudo apt install jq`
- Windows（Git Bash）：從 [https://jqlang.github.io/jq/](https://jqlang.github.io/jq/) 下載 `.exe` 並加入 PATH

驗證：`jq --version` 應輸出版本號。

**錯誤 2：Hook 腳本沒有執行權限**

現象：`PreToolUse` Hook 靜默失敗——`.env` 沒有被保護，操作照常執行，且沒有任何錯誤訊息。

原因：`protect-files.sh` 缺少可執行位元，shell 無法執行它。Hook 系統在腳本無法執行時，某些版本會靜默略過而非報錯。

解決方式：確認已執行 `chmod +x .claude/hooks/protect-files.sh`。用 `ls -la .claude/hooks/` 確認權限列顯示 `-rwxr-xr-x`。

**錯誤 3：ESLint 的非零退出碼導致操作被判定為「阻擋」**

現象：Claude 編輯 `.ts` 檔案後，系統回報操作被阻擋，即使 ESLint 只是發現了 lint 問題而非真正的錯誤。

原因：ESLint 在偵測到 lint 違規時 exit code 為 1，不是 0。如果 PostToolUse Hook 的 `command` 以 ESLint 的 exit code 結束，系統可能誤判為 Hook 執行錯誤。

解決方式：`command` 字串最後必須加 `; exit 0`，強制 Hook 以成功狀態結束。這樣 ESLint 的修正結果會被應用，但不會影響操作的允許/阻擋判斷。

---

### 進階技巧：讓 Linter 錯誤訊息成為修復指令

OpenAI 的 Harness Engineering 實踐揭示了一個被多數人忽略的設計細節：自訂 linter 的錯誤訊息可以專門為 Agent 設計。

傳統 linter 錯誤：
```
error: Function exceeds maximum line count (50)
```

為 Agent 設計的 linter 錯誤：
```
error: Function exceeds 50 lines. Split into smaller functions, each handling
a single responsibility. Extract helper functions to the same module.
Run `npm run lint` to verify after splitting.
```

因為 Hook 的 stderr 輸出會直接回傳給 Claude 作為 context，精心設計的錯誤訊息等於在每次違規時自動注入修復指令。這讓 PostToolUse Hook 不只是「攔截」，而是「攔截 + 教學」。

在 ESLint 中，你可以透過自訂 rule 的 `message` 欄位實現這個效果。投入很小（改幾行錯誤訊息），但對 Agent 的修復成功率影響很大。

---

## 關鍵重點

- CLAUDE.md 規則可靠度 80–95%；Hooks 強制執行，可靠度 100%——兩者不是競爭關係，而是不同層級的工具
- `PreToolUse` 是唯一能阻擋操作的事件；其他事件只能在操作後做處理
- 退出碼是 Hook 與系統唯一的溝通語言：`exit 0` 允許，`exit 2` 阻擋，stderr 訊息告訴 Claude 原因
- `matcher` 讓你精確控制 Hook 的觸發範圍，避免對所有工具造成不必要的延遲
- ESLint 等工具的非零退出碼必須用 `; exit 0` 明確覆寫，否則會產生意外的阻擋行為

---

## 自我檢核

- 如果你想確保「每次 Claude 建立新檔案後，自動在 git 中 stage 它」，你應該用哪個 Hook 事件？matcher 應該設為什麼？
- 當 `PreToolUse` Hook 的腳本執行 `exit 2` 時，Claude 如何知道「為什麼」被阻擋？如果你沒有寫任何 stderr 訊息，會發生什麼？
- CLAUDE.md 規則與 Hooks 各自最適合的場景是什麼？舉出 ai-dev-assistant 中一條應該留在 CLAUDE.md、一條應該升級成 Hook 的規則，並說明理由。
