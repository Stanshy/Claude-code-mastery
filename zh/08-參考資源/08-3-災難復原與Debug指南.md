# 08-3 災難復原與 Debug 指南
> Claude Code 出問題時的系統化排除流程，從 hang 到 crash 到 context 爆炸的完整應對。

## 學習目標
- 能夠依據症狀快速分類故障類型，在 30 秒內決定第一步行動
- 掌握 Claude Code hang、context 爆炸、Hook 錯誤、MCP 連線失敗四大類問題的診斷流程
- 理解 `--resume`、`-c`、`Esc+Esc` 三種回復機制的差異與適用場景
- 建立 Worktree 衝突與回滾的標準處理程序

---

## 正文

### 1. 故障分類與診斷心法

遇到問題時，第一步不是亂試指令，而是分類。以下五種故障涵蓋了 Claude Code 日常使用中 95% 的問題場景：

| 故障類型 | 典型症狀 | 嚴重度 | 第一步行動 |
|---------|---------|--------|-----------|
| Hang / 無回應 | spinner 停轉、終端無輸出超過 60 秒 | 中 | `Ctrl+C` 後觀察回應 |
| Context 爆炸 | 指令遺忘、回應品質驟降、已修正錯誤重現 | 高 | `/cost` 檢查 token 使用率 |
| Hook 錯誤 | 操作被意外阻擋、Hook 靜默失敗、無限迴圈 | 中 | `/hooks` 檢查載入狀態 |
| MCP 連線失敗 | 工具呼叫報錯、timeout、server 狀態 disconnected | 中 | `/mcp` 檢查連線狀態 |
| Worktree 異常 | branch 鎖定、orphan worktree、merge 衝突 | 低 | `git worktree list` 確認狀態 |

**診斷心法**：先分類，再對號入座。不要在 Hook 問題上浪費時間檢查 context，也不要在 context 問題上重啟 MCP server。

---

### 2. Claude Code Hang / 無回應

#### 區分兩種「無回應」

**情境 A：spinner 還在轉，但等了很久**

這不一定是 hang。Claude 在處理大型程式碼庫分析、執行長時間測試、或進行深度推理時，可能需要 2-5 分鐘。判斷標準：

```
spinner 在轉 + 終端標題顯示 "Thinking..." → 等待，至少給 3 分鐘
spinner 在轉 + 終端標題顯示工具名稱（如 "Bash"） → 可能是工具卡住，等 60 秒後考慮中斷
```

**情境 B：spinner 停了，畫面凍結，沒有任何輸出**

這是真正的 hang。處理流程：

```bash
# 第 1 步：嘗試中斷
Ctrl+C

# 如果 Ctrl+C 有回應，Claude 會停止當前操作並等待新輸入
# 你可以繼續對話，或用 /compact 壓縮後重試

# 第 2 步：如果 Ctrl+C 無效，檢查進程
ps aux | grep claude

# 第 3 步：如果進程存在但無回應，強制終止
kill -9 <PID>

# 第 4 步：恢復工作
claude -c          # 繼續最近的會話
claude --resume    # 或從會話選擇器挑選
```

**何時該等，何時該殺**：

| 狀況 | 行動 |
|------|------|
| 正在跑測試（`npm test` / `pytest`） | 等待，測試可能本來就慢 |
| 正在讀取大量檔案 | 等待 2-3 分鐘 |
| 上一次操作是 `git` 命令且卡住 | `Ctrl+C` 中斷，git lock 可能需要手動清理 |
| 畫面完全凍結超過 3 分鐘 | `kill -9` 後用 `claude -c` 恢復 |

#### Git Lock 清理

如果 Claude 在 git 操作中被強制終止，可能留下鎖檔案：

```bash
# 檢查是否有殘留鎖檔
ls .git/*.lock

# 如果存在 index.lock，且確認沒有其他 git 進程在跑
rm .git/index.lock
```

---

### 3. Context 爆炸復原

Context 過載是 Claude Code 使用中最常見且最隱蔽的問題。關於過載的 4 個早期症狀（指令遺忘、已修正錯誤重現、回應變短、成功率下降），詳見 [04-2 上下文管理策略](../04-編排層/04-2-上下文管理策略.md)。

本節聚焦的是：**當 context 已經爆了，如何復原**。

#### 復原決策樹

```
context 使用率超過 80%？
├── 還能理解你的指令 → /compact 搶救
│   └── /compact 後品質恢復 → 繼續工作
│   └── /compact 後品質仍差 → /clear 重啟
└── 已經答非所問 → /clear 重啟
    └── 重啟前先執行：將當前進度寫入檔案
```

#### `/compact` 搶救流程

```bash
# 步驟 1：帶指示壓縮，告訴 Claude 保留什麼
/compact 保留：(1) 當前任務目標 (2) 已完成的步驟 (3) 待處理的問題

# 步驟 2：壓縮後立刻驗證
# 問 Claude 一個只有在記住上下文時才能回答的問題
> 我們目前在修什麼 bug？已經試過哪些方案？
```

#### `/clear` 重啟流程

```bash
# 步驟 1：在 /clear 之前，請 Claude 把進度寫入檔案
> 請把我們目前的進度、已完成項目、待處理項目寫入 PROGRESS.md

# 步驟 2：清除 context
/clear

# 步驟 3：重新載入進度
> 請讀取 PROGRESS.md，然後繼續未完成的任務
```

#### Context 預算分配

這是超出 [04-2](../04-編排層/04-2-上下文管理策略.md) 內容的進階概念。對於 200K token 的 context 窗口，建議的預算分配如下：

| 區塊 | 建議佔比 | 大約 token 數 | 用途 |
|------|---------|-------------|------|
| 系統指令 + CLAUDE.md | 5-10% | 10K-20K | 規則與偏好 |
| 任務描述與背景 | 10-15% | 20K-30K | 當前任務的上下文 |
| 工具呼叫與輸出 | 40-50% | 80K-100K | 檔案讀取、指令輸出 |
| 安全緩衝區 | 20-30% | 40K-60K | 預留給推理空間 |

當「工具呼叫與輸出」佔比超過 60%，回應品質會明顯下降。這時候應該執行 `/compact` 而非繼續塞入更多檔案。

#### `--resume` Checkpoint 復原

如果 Claude Code 意外關閉（terminal crash、系統重啟），你的工作不會遺失：

```bash
# 恢復最近一次會話
claude -c

# 如果最近一次不是你要的，用選擇器
claude --resume
# 會顯示最近的會話列表，選擇要恢復的那個
```

---

### 4. Hook 錯誤除錯

Hook 是系統級攔截器（詳見 [03-1 Hooks 基礎](../03-自動化層/03-1-Hooks基礎.md) 與 [03-2 Hooks 進階](../03-自動化層/03-2-Hooks進階與Validator模式.md)），出錯時的表現往往是靜默的——操作被意外阻擋或該觸發的 Hook 沒有反應。

#### 常見錯誤模式

**模式 1：無限迴圈（Stop Hook）**

症狀：Claude 反覆執行同一個驗證，token 快速消耗，永遠不停止。

原因：Stop Hook 的 Validator 腳本每次都回傳 `exit 1`（要求重試），但 Claude 無法修正問題。

```json
// 錯誤的 Stop Hook — 沒有檢查 stop_hook_active
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "npm test 2>&1; if [ $? -ne 0 ]; then echo 'Tests failed, fix them' >&2; exit 1; fi"
          }
        ]
      }
    ]
  }
}
```

修正：加入 `stop_hook_active` 檢查，且設定重試上限邏輯。

```bash
# 從 stdin 讀取 JSON
INPUT=$(cat)
STOP_HOOK_ACTIVE=$(echo "$INPUT" | jq -r '.stop_hook_active // false')

# 如果已經在 stop hook 重試中，放行避免無限迴圈
if [ "$STOP_HOOK_ACTIVE" = "true" ]; then
  exit 0
fi

npm test 2>&1
if [ $? -ne 0 ]; then
  echo "Tests failed. Please fix the failing tests before completing." >&2
  exit 1
fi
exit 0
```

**模式 2：Matcher 不匹配**

症狀：Hook 已配置但從未觸發。

診斷步驟：

```bash
# 步驟 1：確認 Hook 已載入
# 在 Claude Code 中執行
/hooks
# 檢查輸出中是否列出你的 Hook，以及 matcher 值是否正確

# 步驟 2：確認 matcher 的值
# matcher 匹配的是工具名稱，常見的工具名稱：
# Edit, Write, Bash, Read, Grep, Glob
# 注意大小寫：是 "Edit" 不是 "edit"
```

常見錯誤對照：

| 寫法 | 是否匹配 Edit 工具 | 說明 |
|------|------------------|------|
| `"Edit"` | 匹配 | 精確匹配 |
| `"edit"` | 不匹配 | 大小寫錯誤 |
| `"Edit\|Write"` | 匹配 Edit 和 Write | 用 `\|` 分隔多個工具 |
| `""` | 匹配所有工具 | 空字串是通配 |

**模式 3：腳本執行失敗**

症狀：Hook 有載入、matcher 正確，但行為不符預期。

診斷步驟：

```bash
# 步驟 1：手動執行腳本，模擬 Claude Code 傳入的 JSON
echo '{"tool_name": "Edit", "tool_input": {"file_path": "src/index.ts"}}' | bash .claude/hooks/your-hook.sh

# 步驟 2：檢查退出碼
echo $?
# 0 = 允許, 2 = 阻擋, 其他 = 錯誤

# 步驟 3：檢查 stderr 輸出（這是 Claude 會看到的訊息）
echo '{"tool_name": "Edit", "tool_input": {"file_path": "src/index.ts"}}' | bash .claude/hooks/your-hook.sh 2>&1
```

#### 實際錯誤訊息範例

```
# 錯誤：jq 未安裝
.claude/hooks/protect-files.sh: line 3: jq: command not found
# 修正：安裝 jq（brew install jq / apt install jq）

# 錯誤：權限不足
bash: .claude/hooks/protect-files.sh: Permission denied
# 修正：chmod +x .claude/hooks/protect-files.sh

# 錯誤：JSON 解析失敗
jq: error (at <stdin>:1): null (null) is not iterable
# 修正：檢查 jq 的 path 表達式，用 // empty 處理 null 值
```

---

### 5. MCP Server 連線失敗

MCP 讓 Claude Code 連接外部系統（詳見 [03-4 MCP Servers](../03-自動化層/03-4-MCP-Servers.md)）。連線問題是最常見的 MCP 故障。

#### 常見原因與診斷

**原因 1：Token 過期或無效**

```bash
# 在 Claude Code 中檢查 MCP 狀態
/mcp
# 觀察各 server 的狀態：connected / disconnected / error

# 如果 GitHub MCP 顯示 error，通常是 token 問題
# 重新設定 token
claude mcp add github --url https://api.githubcopilot.com/mcp/ --header "Authorization: Bearer YOUR_NEW_TOKEN"
```

**原因 2：Port 衝突（stdio 類型的本地 MCP server）**

```bash
# 檢查是否有殘留的 MCP server 進程
ps aux | grep mcp

# 如果有殘留進程佔用 port
kill <PID>

# 重啟 Claude Code 會自動重新啟動 MCP server
```

**原因 3：`.mcp.json` 配置格式錯誤**

```bash
# 驗證 JSON 格式
node -e "console.log(JSON.parse(require('fs').readFileSync('.mcp.json','utf8')))"

# 常見格式錯誤：
# - 多餘的逗號（trailing comma）
# - env 值不是字串
# - args 不是陣列
```

正確的 `.mcp.json` 格式：

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/",
      "headers": {
        "Authorization": "Bearer ghp_xxxxxxxxxxxx"
      }
    },
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-filesystem"],
      "env": {
        "ALLOWED_DIRS": "/home/user/projects"
      }
    }
  }
}
```

#### MCP 手動測試

當 `/mcp` 顯示連線失敗時，繞過 Claude Code 直接測試：

```bash
# 測試 HTTP 類型的 MCP server
curl -s -H "Authorization: Bearer YOUR_TOKEN" https://api.githubcopilot.com/mcp/ -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'

# 如果回傳 401 → token 無效
# 如果回傳 tools 列表 → MCP server 正常，問題在 Claude Code 配置
# 如果 connection refused → server 不可達
```

---

### 6. Worktree 衝突與回滾

Worktree 是平行開發的基礎（詳見 [04-3 平行開發與 Worktree](../04-編排層/04-3-平行開發與Worktree.md)）。以下是 Worktree 出問題時的標準處理程序。

#### 常見問題

**問題 1：Orphan Worktree（工作目錄已刪但記錄殘留）**

```bash
# 列出所有 worktree
git worktree list
# 輸出示例：
# /home/user/project           abc1234 [main]
# /home/user/project-feature-x def5678 [feature-x]  ← 如果目錄已不存在

# 清理無效的 worktree 記錄
git worktree prune

# 驗證
git worktree list
```

**問題 2：Branch 被鎖定（另一個 worktree 正在使用）**

```bash
# 錯誤訊息：
# fatal: 'feature-x' is already checked out at '/home/user/project-feature-x'

# 解法 1：如果那個 worktree 已經不需要
git worktree remove /home/user/project-feature-x

# 解法 2：如果需要在新位置使用同一個 branch
git worktree remove /home/user/project-feature-x
git worktree add /home/user/project-feature-x-v2 feature-x
```

**問題 3：Worktree 中的 Merge 衝突**

```bash
# 在 worktree 目錄中操作
cd /path/to/worktree

# 查看衝突檔案
git status
# 輸出示例：
# both modified:   src/utils.ts

# 選項 A：接受 main 的版本
git checkout --theirs src/utils.ts

# 選項 B：接受 worktree branch 的版本
git checkout --ours src/utils.ts

# 選項 C：手動解決後
git add src/utils.ts
git commit -m "resolve merge conflict in src/utils.ts"
```

#### Worktree 完整清理

當你完成一個功能分支的開發，標準清理流程：

```bash
# 步驟 1：確認 worktree 中的變更已 commit 或 push
cd /path/to/worktree
git status  # 確認 working tree clean

# 步驟 2：回到主 repo
cd /path/to/main-repo

# 步驟 3：移除 worktree
git worktree remove /path/to/worktree

# 步驟 4：如果 branch 已合併，可以刪除
git branch -d feature-x

# 步驟 5：清理任何殘留
git worktree prune
```

---

### 7. Checkpoint 與 --resume 完整用法

Claude Code 提供三種回到先前狀態的機制，各自適用不同場景。

#### Esc+Esc 回滾選單

按兩次 `Esc` 鍵，Claude Code 會顯示 Checkpoint 回滾選單。這個功能讓你撤銷 Claude 最近的檔案變更。

```
操作流程：
1. 按 Esc+Esc → 出現 Checkpoint 列表
2. 每個 Checkpoint 對應一次工具操作（如 Edit、Write、Bash）
3. 選擇要回滾到的 Checkpoint → Claude 撤銷該操作之後的所有檔案變更
4. 對話歷史保留，但檔案狀態回到選定的 Checkpoint
```

適用場景：Claude 剛做了一個錯誤的修改，你想撤銷檔案變更但保留對話上下文。

#### claude --resume 會話選擇器

```bash
claude --resume
# 顯示最近的會話列表，包含：
# - 會話 ID
# - 開始時間
# - 最後一則訊息摘要
# 選擇後恢復該會話的完整上下文
```

適用場景：你有多個會話，需要回到特定的某一個。

#### claude -c 繼續最近會話

```bash
claude -c
# 直接恢復最近一次會話，不顯示選擇器
```

適用場景：Claude Code 意外關閉，你想快速回到剛才的工作。

#### 三種機制對比

| 機制 | 觸發方式 | 回滾範圍 | 對話歷史 | 適用場景 |
|------|---------|---------|---------|---------|
| `Esc+Esc` | 鍵盤快捷鍵 | 檔案變更 | 保留 | 撤銷錯誤的檔案修改 |
| `--resume` | CLI 參數 | 整個會話 | 恢復 | 切換到特定的歷史會話 |
| `-c` | CLI 參數 | 最近會話 | 恢復 | 意外中斷後快速恢復 |

#### 組合使用範例

```bash
# 場景：Claude 在修改 3 個檔案後把程式碼搞壞了
# 步驟 1：按 Esc+Esc，回滾到修改前的 Checkpoint
# 步驟 2：告訴 Claude 用不同的方式重試
> 剛才的做法行不通。改用 X 方式實作。

# 場景：terminal 被意外關閉
# 步驟 1：重開 terminal
claude -c
# 步驟 2：確認上下文完整
> 我們剛才進行到哪裡了？

# 場景：需要回到昨天的某個會話
claude --resume
# 從列表中選擇昨天的會話
```

---

## 實作範例：模擬 Hook 錯誤並除錯修復

**目標**：故意製造一個有 bug 的 PreToolUse Hook，觀察錯誤症狀，使用本章的診斷流程定位並修復問題。

**前置條件**：
- 已完成 [03-1 Hooks 基礎](../03-自動化層/03-1-Hooks基礎.md) 的實作範例（ai-dev-assistant 專案已有基礎 Hook 配置）
- 系統已安裝 `jq`（`jq --version` 可正常輸出）

**步驟**：

**步驟 1**：在 ai-dev-assistant 專案中，建立一個故意寫錯的 Hook 腳本 `.claude/hooks/buggy-guard.sh`：

```bash
#!/bin/bash
# 故意的 bug：jq path 表達式寫錯，會導致解析失敗
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.filePath')
# 正確的 key 是 .tool_input.file_path（底線），不是 .filePath（駝峰）

if [[ "$FILE_PATH" == *.test.ts ]]; then
  echo "Blocked: do not modify test files directly" >&2
  exit 2
fi

exit 0
```

**步驟 2**：設定腳本權限：

```bash
chmod +x .claude/hooks/buggy-guard.sh
```

**步驟 3**：在 `.claude/settings.json` 中加入這個有 bug 的 Hook：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/buggy-guard.sh"
          }
        ]
      }
    ]
  }
}
```

**步驟 4**：在 Claude Code 中請 Claude 編輯一個 `.test.ts` 檔案（例如 `src/index.test.ts`）。觀察結果——Hook 應該要阻擋，但因為 jq path 寫錯，`FILE_PATH` 會是 `null`，條件判斷不成立，操作會被放行。

**步驟 5**：執行診斷流程。先在 Claude Code 中執行 `/hooks` 確認 Hook 已載入且 matcher 正確。然後手動測試腳本：

```bash
echo '{"tool_name": "Edit", "tool_input": {"file_path": "src/index.test.ts"}}' | bash .claude/hooks/buggy-guard.sh
echo $?
# 輸出 0（放行），而預期應該是 2（阻擋）
```

**步驟 6**：用加入 debug 輸出的方式定位 bug：

```bash
echo '{"tool_name": "Edit", "tool_input": {"file_path": "src/index.test.ts"}}' | jq -r '.tool_input.filePath'
# 輸出：null

echo '{"tool_name": "Edit", "tool_input": {"file_path": "src/index.test.ts"}}' | jq -r '.tool_input.file_path'
# 輸出：src/index.test.ts
```

**步驟 7**：修正 `.claude/hooks/buggy-guard.sh`，將 `.tool_input.filePath` 改為 `.tool_input.file_path`：

```bash
#!/bin/bash
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

if [[ "$FILE_PATH" == *.test.ts ]]; then
  echo "Blocked: do not modify test files directly" >&2
  exit 2
fi

exit 0
```

**步驟 8**：驗證修正後的腳本：

```bash
echo '{"tool_name": "Edit", "tool_input": {"file_path": "src/index.test.ts"}}' | bash .claude/hooks/buggy-guard.sh
echo $?
# 輸出 2（阻擋）

echo '{"tool_name": "Edit", "tool_input": {"file_path": "src/index.ts"}}' | bash .claude/hooks/buggy-guard.sh
echo $?
# 輸出 0（放行）
```

**步驟 9**：回到 Claude Code，再次請 Claude 編輯 `.test.ts` 檔案。這次操作應被阻擋，Claude 應收到 "Blocked: do not modify test files directly" 訊息。

**預期結果**：
- 步驟 4 中，有 bug 的 Hook 靜默放行了本應阻擋的操作——這就是 Hook 錯誤的典型隱蔽性
- 步驟 5-6 的手動腳本測試精準定位了 jq path 表達式的錯誤
- 修正後，Hook 正確阻擋 `.test.ts` 檔案的編輯，且不影響其他檔案的正常編輯
- Claude 收到 stderr 中的阻擋訊息，並會調整策略（如改為建議你手動修改測試檔案）

**常見錯誤**：

**錯誤 1：手動測試時忘記用 pipe 傳入 JSON**

現象：直接執行 `bash .claude/hooks/buggy-guard.sh`，腳本卡住不動。

原因：腳本第一行 `INPUT=$(cat)` 在等待 stdin 輸入。沒有 pipe 輸入時，`cat` 會阻塞等待直到你按 `Ctrl+D`。

解決方式：測試 Hook 腳本時，一定要用 `echo '{"tool_name": "Edit", "tool_input": {"file_path": "test.ts"}}' | bash your-hook.sh` 的格式，模擬 Claude Code 傳入的 JSON。

**錯誤 2：修正腳本後忘記重啟 Claude Code**

現象：修正了 Hook 腳本，但 Claude Code 的行為沒有變化。

原因：部分版本的 Claude Code 會快取 Hook 配置。修改 `.claude/settings.json` 的結構（新增或刪除 Hook 項目）通常需要重啟才能生效。修改已引用的外部腳本檔案內容則通常不需要重啟，因為每次觸發時會重新讀取腳本。

解決方式：如果修改後行為未變，退出 Claude Code 後用 `claude -c` 重新進入。用 `/hooks` 確認配置已更新。

**錯誤 3：jq 的 `// empty` 被遺漏，導致後續字串比對行為異常**

現象：Hook 腳本在某些情況下對不存在的欄位回傳 `"null"` 字串（不是空字串），導致條件判斷出現不可預期的匹配。

原因：`jq -r '.tool_input.file_path'` 在欄位不存在時輸出字面量 `null`。如果你的條件是 `[[ "$FILE_PATH" == *something* ]]`，`null` 不會匹配，看似正常。但如果條件更寬鬆（例如 `-n` 非空判斷），`"null"` 會被當作有值。

解決方式：使用 `jq -r '.tool_input.file_path // empty'`，在值為 null 時輸出空字串而非 `"null"`。

---

## 關鍵重點

- 故障診斷的第一步是分類，不是亂試指令——用症狀表對照定位問題類型，再走對應的診斷流程
- Hang 的兩種情境（spinner 在轉 vs 完全凍結）處理方式截然不同：前者等待，後者 `Ctrl+C` 或 `kill`
- Context 爆炸的復原有兩條路徑：`/compact` 搶救（保留上下文）和 `/clear` 重啟（先寫 PROGRESS.md 再清除），根據 Claude 是否還能理解指令來選擇
- Hook 錯誤最危險的不是報錯，而是靜默失敗——該觸發的沒觸發，你以為安全但其實沒有防護。手動 pipe JSON 測試腳本是最可靠的驗證方式
- `Esc+Esc`（回滾檔案）、`--resume`（會話選擇器）、`-c`（最近會話）三個機制解決不同問題，不要混用
- Worktree 問題的根源通常是殘留記錄或 branch 鎖定，`git worktree prune` 是第一個該跑的清理命令

---

## 自我檢核

- Claude Code 在執行 `npm test` 時已經跑了 4 分鐘沒有回應，spinner 還在轉。你應該立刻 `kill -9` 還是繼續等待？你會用什麼資訊來做判斷？
- 你發現 Claude 開始忘記你在 session 開頭設定的命名規則，而且回應越來越簡短。你判斷這是什麼問題？你的前兩步行動是什麼？如果第一步沒有改善，你的 fallback 方案是什麼？
- 你配置了一個 PreToolUse Hook 來阻擋 `.env` 的修改，但 Claude 仍然成功編輯了 `.env.local`。按照本章的診斷流程，你應該依序檢查哪三件事？
- 你在 worktree 中完成了功能開發，想切回 main branch 繼續工作，但 `git checkout main` 回報 `fatal: 'main' is already checked out`。這是什麼原因？你的解決步驟是什麼？


---

⬅️ [上一章：資源索引](08-2-資源索引.md) ｜ 📖 [回目錄](../00-索引與摘要.md)

🌐 [English Version](../../en/08-reference/08-3-disaster-recovery.md)
