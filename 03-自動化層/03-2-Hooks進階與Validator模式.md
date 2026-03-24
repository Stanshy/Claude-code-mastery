# 03-2 Hooks 進階與 Validator 模式
> 如果你只學一個進階能力，就是這章。Stop Hook Validator Pattern 是讓 Claude Code 從「可能出錯」變成「保證正確」的關鍵機制。

## 學習目標
- 理解 Stop Hook Validator Pattern 的數學原理，以及它為何是最重要的設計模式
- 掌握 `stop_hook_active` 欄位的用途，避免無限迴圈
- 認識四種 Hook 處理器類型（command、prompt、agent、http）及其適用場景
- 學會組合多個 Hook 形成串聯驗證管線

---

## 正文

### 1. 為什麼 Validator 是最重要的設計模式

David Chu 的核心洞察值得直接引用：**好的驗證器搭配差的工作流程，勝過好的工作流程但沒有驗證器。**

這句話看起來反直覺，但數學完全支持它。

假設你有一個 5 個步驟的自動化工作流程，每個步驟的成功率是 80%。整體成功率是：

```
0.8 × 0.8 × 0.8 × 0.8 × 0.8 = 0.8^5 ≈ 33%
```

你花了大量時間打磨工作流程，最終三分之二的執行結果都是失敗的。

現在換個角度：加入 Stop Hook Validator，即使每個單步的可靠度只有 70%，但允許 4 次重試。成功率變為：

```
1 - 0.3^4 = 1 - 0.0081 ≈ 99%
```

驗證器的數學本質是：**它把「串聯失敗」變成「並聯重試」**。串聯系統中任何一個環節失敗就全盤皆輸；並聯重試系統中，只要有一次成功就算過關。這個結構性的差異，才是為什麼驗證器的優先級高於工作流程最佳化。

---

### 2. Stop Hook Validator Pattern

Stop Hook 是所有 Hook 類型中最具戰略價值的一個。它攔截的是「Claude 認為任務已完成，準備停止」這個時機點。

**工作原理（完整流程）：**

1. Claude 執行完任務，準備停止
2. Stop Hook 攔截，執行驗證（例如 `npm test` + `npm run lint`）
3. 如果驗證失敗 → 失敗訊息回傳給 Claude → Claude 自動修復 → 再次嘗試停止
4. 如果驗證通過 → 允許停止，任務結束

**關鍵機制：`stop_hook_active` 欄位**

這是最容易被忽略、也最危險的細節。當 Stop Hook 觸發後，Claude 修復問題並再次嘗試停止時，Claude Code 會在傳入 Hook 的 JSON input 中加入 `stop_hook_active: true`。你的 Hook 腳本**必須**在偵測到這個欄位時直接 `exit 0`，否則 Hook 會再次觸發驗證，形成無限迴圈。

完整的防迴圈 Stop Validator 腳本如下：

```bash
#!/bin/bash
INPUT=$(cat)

# 防止無限迴圈：如果已經在 stop hook 中，直接放行
STOP_ACTIVE=$(echo "$INPUT" | jq -r '.stop_hook_active // false')
if [ "$STOP_ACTIVE" = "true" ]; then
  exit 0
fi

# 執行驗證
cd "$CLAUDE_PROJECT_DIR"
npm test 2>&1
TEST_RESULT=$?

npm run lint 2>&1
LINT_RESULT=$?

if [ $TEST_RESULT -ne 0 ] || [ $LINT_RESULT -ne 0 ]; then
  echo "Validation failed. Fix the issues before completing." >&2
  exit 2
fi

exit 0
```

腳本中有三個設計決策值得說明：

- `cd "$CLAUDE_PROJECT_DIR"`：確保 npm 指令在正確的專案根目錄執行，否則找不到 `node_modules`
- `echo ... >&2`：錯誤訊息必須寫入 stderr，Claude Code 才能讀取並回傳給 Claude；stdout 留給 JSON 通訊用途
- `exit 2`：exit code 2 代表「阻擋操作並回傳錯誤訊息」；exit 1 代表「阻擋但不回傳訊息」；exit 0 代表「允許繼續」

---

### 3. 四種 Hook 處理器類型

03-1 章節只介紹了 `command` 類型。完整的 Hook 處理器有四種，各自解決不同層次的驗證需求：

| 類型 | 說明 | 適用場景 |
|------|------|---------|
| `command` | 執行 shell 命令 | 格式化、檔案保護、驗證腳本 |
| `prompt` | 用 Haiku 模型做單輪語義判斷 | 語義檢查（如「這個變更是否符合規範」） |
| `agent` | 用子代理做多輪驗證 | 複雜驗證（跑測試、檢查覆蓋率、code review） |
| `http` | POST JSON 到 HTTP 端點 | 外部系統整合（審計日誌、Slack 通知） |

**`prompt` 類型範例**

適合需要語義理解、但不需要執行程式的情境。例如確認每個修改的檔案都有對應的測試檔：

```json
{
  "type": "prompt",
  "prompt": "Check if all modified files have corresponding test files. Return {\"ok\": true} or {\"ok\": false, \"reason\": \"...\"}"
}
```

這個類型使用的是 Claude Haiku 模型，速度快且成本低，適合高頻觸發的事件（如 PostToolUse）。

**`agent` 類型範例**

適合需要多步驟推理或執行多個工具的複雜驗證。例如執行完整測試套件並分析結果：

```json
{
  "type": "agent",
  "prompt": "Run the full test suite and verify all tests pass. If any fail, report which tests failed and why.",
  "timeout": 120
}
```

`timeout` 欄位（單位：秒）是必要的，因為測試套件可能執行時間較長。沒有設定 timeout 的 agent hook 可能讓整個流程懸掛。

**`http` 類型範例**

適合需要整合外部系統的情境。每次 Hook 觸發時，Claude Code 會將事件資料以 POST JSON 方式送出：

```json
{
  "type": "http",
  "url": "http://localhost:8080/hooks/audit",
  "headers": {
    "Authorization": "Bearer $TOKEN"
  },
  "allowedEnvVars": ["TOKEN"]
}
```

`allowedEnvVars` 是安全機制：只有在這個白名單中的環境變數才會被展開。直接把密鑰硬寫在 JSON 中是錯誤做法。

---

### 4. 組合 Hook 的設計模式

同一個 Hook 事件可以配置多個 Hook 處理器，形成串聯管線。每個處理器依序執行，任何一個 `exit 2` 都會阻擋操作並回傳錯誤。

**PostToolUse 串聯範例：**
- 第一個 Hook：格式化（Prettier）
- 第二個 Hook：Lint 檢查（ESLint）
- 第三個 Hook：記錄操作日誌（http 類型）

**Stop 串聯範例：**
- 第一個 Hook：跑測試（`npm test`）
- 第二個 Hook：型別檢查（`npm run type-check`）
- 第三個 Hook：檢查測試覆蓋率是否達標

設計原則：**越快的檢查放越前面**。格式化和 lint 只需要幾秒，放在最前面可以在耗時的測試之前快速過濾明顯問題，節省整體執行時間。

---

### 5. Hook 故障排除

Hook 的問題通常不容易直接看到，因為很多失敗是靜默的。以下是最常見的五個問題及對應解法：

| 問題 | 原因 | 解法 |
|------|------|------|
| Hook 未觸發 | matcher 拼寫錯誤 | 用 `/hooks` 指令確認設定；注意 matcher 區分大小寫 |
| JSON parse 失敗 | `.bashrc` 或 `.zshrc` 中有多餘的 `echo` 輸出 | 確保 shell profile 在非互動模式下不輸出任何非 JSON 內容 |
| Hook 超時 | 腳本執行時間超過預設限制 | 在設定中加入 `timeout` 欄位（單位：秒）；優化腳本效能 |
| 無限迴圈 | Stop Hook 沒有檢查 `stop_hook_active` | 必須在 `stop_hook_active = true` 時執行 `exit 0` |
| Hook 靜默失敗 | 腳本沒有執行權限 | 對所有 hook 腳本執行 `chmod +x` |

其中「JSON parse 失敗」是最隱蔽的問題。Claude Code 與 Hook 腳本之間透過 stdout 傳遞 JSON，如果 shell profile（`.bashrc`、`.zshrc`）在啟動時輸出任何文字（例如 `echo "Welcome"` 或一些工具的初始化訊息），這些文字會混入 stdout，導致 JSON 解析失敗，但你幾乎看不到任何錯誤提示。

---

## 實作範例：為 AI Dev Assistant 加入 Stop Validator

> 主案例延續：在 AI Dev Assistant 專案中，我們已完成基礎 Hook 設定（格式化與檔案保護）。現在要加入 Stop Hook，確保 Claude 在任何情況下都無法在測試失敗的狀態下結束任務。

**目標**：加入 Stop Hook，讓 Claude 在完成任務前必須通過 `npm test` 和 `npm run lint`

**前置條件**：
- 已完成 03-1 章節（`.claude/settings.json` 已存在，基礎 Hook 已設定）
- 專案根目錄為 `/projects/ai-dev-assistant`
- `package.json` 中已有 `test` 和 `lint` 腳本

**步驟：**

1. 在 `.claude/hooks/` 目錄下建立驗證腳本 `stop-validator.sh`，內容使用本章正文第 2 節提供的完整腳本（含 `stop_hook_active` 防護邏輯）。

2. 賦予腳本執行權限：
   ```bash
   chmod +x /projects/ai-dev-assistant/.claude/hooks/stop-validator.sh
   ```

3. 開啟 `.claude/settings.json`，在 `hooks` 物件中加入 `Stop` 事件：
   ```json
   "Stop": [
     {
       "hooks": [
         {
           "type": "command",
           "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/stop-validator.sh"
         }
       ]
     }
   ]
   ```
   注意 `command` 路徑使用 `$CLAUDE_PROJECT_DIR` 環境變數，確保在任何工作目錄下都能正確定位腳本。

4. 儲存 `settings.json`，然後啟動 Claude Code：
   ```bash
   cd /projects/ai-dev-assistant && claude
   ```

5. 給 Claude 一個會導致測試失敗的任務：
   ```
   在 src/index.ts 的 main 函數中加一個 bug：把回傳值改為 undefined
   ```

6. 觀察執行流程：Claude 完成修改後嘗試停止 → Stop Hook 觸發 → `npm test` 失敗 → Claude 收到 stderr 訊息 → Claude 自動修復 bug → 再次嘗試停止 → Stop Hook 再次觸發 → `stop_hook_active` 為 true → 直接 `exit 0` → 任務正常結束。

7. 在終端機輸出中確認 Stop Hook 的觸發記錄，並執行 `npm test` 確認所有測試通過。

**預期結果**：
- Claude 無法在測試失敗的狀態下結束任務，即使 Claude「認為」自己已經完成
- Claude 會自動偵測驗證失敗並進行修復，不需要人工介入
- 終端機中可以看到兩次（或以上）的 Stop Hook 觸發記錄
- 最終的程式碼狀態是測試全數通過、lint 無警告

**常見錯誤：**

- **無限迴圈（Claude 持續嘗試停止但一直被擋）**：原因幾乎必然是腳本中缺少 `stop_hook_active` 的檢查。確認腳本第一段邏輯正確讀取並判斷這個欄位。用 `echo "$INPUT" | jq '.'` 在腳本中偵錯，確認 JSON input 的實際結構。

- **`npm test` 在 Hook 中找不到 `node_modules`**：Hook 腳本的工作目錄不一定是專案根目錄。腳本中必須有 `cd "$CLAUDE_PROJECT_DIR"` 這一行。確認 `$CLAUDE_PROJECT_DIR` 環境變數的值是否正確（可在腳本中加 `echo "$CLAUDE_PROJECT_DIR" >&2` 偵錯）。

- **Hook 執行了但 Claude 沒有收到錯誤訊息**：確認錯誤訊息是寫到 stderr（`echo "..." >&2`），而不是 stdout。stdout 是 Claude Code 用於 JSON 通訊的通道，任何非 JSON 的 stdout 輸出都可能造成解析問題，不會被當作錯誤訊息處理。

---

## 關鍵重點
- Validator 的核心價值在於把串聯失敗改為並聯重試，即使單步可靠度較低，整體可靠度仍可接近 100%
- `stop_hook_active: true` 是防止 Stop Hook 無限迴圈的唯一機制，所有 Stop Validator 腳本都必須處理這個欄位
- 四種 Hook 類型（command、prompt、agent、http）對應四種不同的驗證複雜度，選擇時依據「能用 command 解決就不用 agent」的最小複雜度原則
- 多個 Hook 串聯時，效能排序原則是：快速的檢查（格式化、lint）放前面，耗時的檢查（測試、覆蓋率）放後面
- 錯誤訊息必須寫到 stderr，stdout 保留給 JSON 通訊；這個規則不遵守會導致靜默失敗，極難偵錯

## 自我檢核
- 如果你的 Stop Hook 腳本沒有 `stop_hook_active` 的檢查，實際執行時會發生什麼事？能描述完整的迴圈過程嗎？
- 在一個有 PostToolUse 事件的 Hook 配置中，你會如何決定使用 `command` 類型還是 `prompt` 類型？各自的判斷標準是什麼？
- 承接上一章的 `ai-dev-assistant` 設定，現在的 `settings.json` 應該同時有 `PostToolUse` 和 `Stop` 兩個事件的 Hook。如果兩個事件都有 lint 檢查，你會如何重構避免重複？
