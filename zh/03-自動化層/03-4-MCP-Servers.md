# 03-4 MCP Servers
> MCP（Model Context Protocol）讓 Claude Code 突破本地檔案系統的邊界，直接與 GitHub、資料庫、Slack 等外部系統互動，無需手動撰寫整合腳本。

## 學習目標
- 說明 MCP 解決的根本問題及其與手動 API 呼叫的差異
- 掌握 CLI 與 `.mcp.json` 兩種安裝方式及各自適用場景
- 理解三種作用域（專案、使用者、當次會話）的優先順序
- 成功連接 GitHub MCP 並驗證 Claude 能直接查詢 repo 資料

---

## 正文

### 1. 為什麼需要 MCP

Claude Code 預設只能操作兩類資源：本地檔案系統和 shell 指令。這個設計在大多數情況下已夠用，但真實開發中你需要與外部系統互動：

- 讀取 GitHub Issue，理解需求背景
- 查詢 PostgreSQL，分析資料分佈
- 透過 Slack 發送部署通知
- 用瀏覽器截圖驗證 UI 改動

沒有 MCP 時，傳統做法是：手動執行 `curl` → 複製回應 → 貼入 Claude Code → Claude 分析 → 需要追問時再重複一遍。這個流程的根本問題是**資料是靜態的**，Claude 無法自主發起後續查詢。

MCP（Model Context Protocol）是一個開放標準，定義了 Claude 與外部工具伺服器之間的通訊協議。安裝 MCP server 後，Claude 能直接呼叫外部 API，就像呼叫本地的 Bash 工具一樣——包括自主決定何時需要查詢、查詢什麼內容、根據結果繼續推進任務。

類比：Hook 是 Claude Code 的「自動觸發器」，Skill 是「工作手冊」，MCP 是「外接工具箱」。

---

### 2. MCP 的工作原理

MCP Server 是一個獨立運行的程式，暴露一組具名工具給 Claude Code 使用。Claude Code 透過兩種方式與它溝通：

| 類型 | 通訊方式 | 適用場景 |
|------|---------|---------|
| `stdio` | stdin / stdout | 本地程式（如 npx 啟動的 server） |
| `http` | HTTP / HTTPS | 遠端 API（如 GitHub 官方 MCP 端點） |

連線建立後，Claude 在需要外部資料時會呼叫 MCP 工具，MCP server 執行實際的 API 請求並將結果回傳給 Claude。整個流程對使用者透明——你看到的是 Claude 直接提供結果，而不是讓你手動複製貼上。

---

### 3. 安裝方式

MCP server 有兩種安裝方式，對應不同的使用場景。

**方式一：CLI 指令**

```bash
claude mcp add github https://api.github.com/mcp/ --scope user
```

CLI 方式快速，適合快速設定官方知名 MCP server。`--scope user` 讓設定對所有專案生效，省略則預設為專案級別（local scope）。

**方式二：配置檔 `.mcp.json`**

在專案根目錄建立 `.mcp.json`：

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

配置檔方式的優勢是可以進入版本控制，讓整個團隊使用相同的 MCP 設定。注意 `$GITHUB_TOKEN` 這類敏感資訊必須透過環境變數注入，**不要直接寫入配置檔**。

---

### 4. 作用域

MCP server 的作用域決定在哪些情境下生效：

| 位置 | 範圍 | 適用場景 |
|------|------|---------|
| 專案 `.mcp.json` | 僅當前專案 | 專案特定工具、需要團隊共享的設定 |
| `~/.claude/.mcp.json` | 當前使用者的所有專案 | 個人常用工具（GitHub、Slack） |
| `--mcp-config` CLI 參數 | 當次會話 | 臨時測試新的 MCP server |

當多個作用域定義了相同名稱的 MCP server 時，**專案級別的 `.mcp.json` 優先權最高**，可覆蓋使用者級別的設定。這允許特定專案使用不同版本或不同端點的相同服務。

---

### 5. Tool Search 自動模式

啟用多個 MCP server 有一個容易忽略的 context 成本問題。

每個 MCP server 連線時需要向 Claude 描述自己提供的所有工具。當你同時啟用 5 個 MCP server，每個有 10 幾個工具時，光是工具描述就可能消耗約 55,000 tokens。這個開銷在每次會話開始時都會發生，即使那次任務只用到其中一個工具。

Tool Search 採用「按需發現」機制：Claude 不在會話開始時載入所有工具描述，而是在需要時搜尋相關工具。這個機制可將 context 消耗降低約 85%（約 8,700 tokens）。

管理多個 MCP server 的建議：
- 只啟用當前任務實際需要的 MCP server
- 使用頻率低的 server 優先使用 `--mcp-config` 以 session 為單位啟用
- 定期執行 `/mcp` 確認哪些 server 處於 connected 狀態

---

### 6. 常見 MCP Server 推薦

| Server | 用途 | 安裝指令 |
|--------|------|---------|
| GitHub | repo / PR / issue 操作 | `claude mcp add github https://api.github.com/mcp/` |
| Playwright | 瀏覽器自動化 / 截圖 | `claude mcp add playwright --type stdio npx @anthropic-ai/mcp-playwright` |
| PostgreSQL | 資料庫查詢 | `claude mcp add postgres --type stdio npx @anthropic-ai/mcp-postgres` |

---

### 7. 什麼時候不該用 MCP

MCP 不是萬能解法，以下情境直接用 shell 更合適：

**只需一次性的簡單 API 呼叫**：從 GitHub 抓一次某個檔案內容，直接執行 `curl` 更快，不需要安裝和配置整個 MCP server。

**高安全性場景**：MCP server 以你的身份操作外部系統，具有寫入權限時風險尤其高。對於 Anthropic 官方以外的第三方 MCP server，安裝前應審查其原始碼。

MCP 的價值在於需要**多步驟、動態互動**的任務——Claude 需要根據第一次查詢的結果決定下一步查詢什麼。一次性操作不值得引入這層複雜度。

---

## 實作範例：為 AI Dev Assistant 接上 GitHub MCP
> 主案例延續：在 AI Dev Assistant 專案中，開發者需要頻繁查看 GitHub Issue 來理解功能需求，目前都是手動複製 API 回應再貼入 Claude。我們接上 GitHub MCP，讓 Claude 能直接讀 Issue、搜尋 PR，消除這個手動橋接步驟。

**目標**：設定 GitHub MCP Server，讓 Claude 能直接讀 Issue、建 PR，驗證多步驟查詢能自動串聯

**前置條件**：
- 已完成 02-2（settings.json 配置）
- 有 GitHub 帳號
- 已建立 GitHub Personal Access Token（需要 `repo` 與 `read:org` 權限）

**步驟**：

**步驟 1**：取得 GitHub Personal Access Token

前往 GitHub Settings → Developer settings → Personal access tokens → Fine-grained tokens，建立新 token，選擇需要存取的 repositories，權限至少包含 `Contents: Read` 與 `Issues: Read and Write`。

**步驟 2**：設定環境變數

```bash
export GITHUB_TOKEN=ghp_xxxxx
```

建議將此行加入 `~/.bashrc` 或 `~/.zshrc` 使其永久生效。更持久的方式是加入 `.claude/settings.local.json` 的 `env` 欄位：

```json
{
  "env": {
    "GITHUB_TOKEN": "ghp_xxxxx"
  }
}
```

注意：`settings.local.json` 不應進入版本控制，確認 `.gitignore` 已包含此檔案。

**步驟 3**：在專案根目錄建立 `.mcp.json`

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

**步驟 4**：確認 `GITHUB_TOKEN` 環境變數已設定

```bash
echo $GITHUB_TOKEN
```

應顯示以 `ghp_` 或 `github_pat_` 開頭的 token。如果為空，先回步驟 2。

**步驟 5**：啟動 Claude Code

```bash
claude
```

**步驟 6**：用 `/mcp` 確認 GitHub server 已連線

```
/mcp
```

應看到：

```
MCP Servers:
  github    connected
```

如果顯示 `disconnected` 或 `error`，先排除認證問題再繼續（見常見錯誤）。

**步驟 7**：執行第一個 MCP 查詢

輸入：

```
列出我的 GitHub repos 中最近更新的 5 個
```

觀察 Claude 的行為：它會顯示正在呼叫 GitHub MCP 工具，然後直接回傳結構化的 repo 清單，包含名稱、最後更新時間和語言。整個過程不需要手動執行 `curl` 或複製貼上。

**步驟 8**：測試多步驟查詢（驗證 MCP 的核心價值）

輸入：

```
查看 ai-dev-assistant repo 的最新 3 個 open issue，告訴我哪個優先度最高
```

觀察 Claude 如何：先查詢 issue 清單、讀取每個 issue 的詳細內容、根據標籤和描述判斷優先度，最後給出建議。這個查詢需要多次 MCP 呼叫，且每次呼叫的參數取決於上一次的結果——這正是 MCP 比手動 API 呼叫有本質優勢的場景。

**預期結果**：
- `/mcp` 顯示 github server 狀態為 connected
- Claude 能讀取你的 GitHub repo 列表，不需要手動執行 `curl`
- 多步驟查詢（如「找出優先度最高的 issue」）能自動串聯多次 API 呼叫並整合結果

**常見錯誤**：

**錯誤 1：`401 Unauthorized` 或 `/mcp` 顯示 disconnected**

原因：Token 無效、過期，或 `$GITHUB_TOKEN` 環境變數未正確設定。

解決步驟：
1. 執行 `echo $GITHUB_TOKEN` 確認有值
2. 在 GitHub Settings 確認 token 未過期，且 scope 包含 `repo`
3. 若 token 正確但仍失敗，確認 `.mcp.json` 的 JSON 格式無誤（用 `cat .mcp.json | python3 -m json.tool` 驗證）
4. 重新啟動 Claude Code

**錯誤 2：`/mcp` 顯示 connected 但查詢無回應或回傳空結果**

原因：Token 的權限不足，或 Fine-grained token 未選擇正確的 repository 範圍。

解決步驟：
1. 確認 token 的 `Repository access` 包含目標 repo（或選擇 All repositories）
2. 確認 token 權限包含 `Contents: Read` 和 `Issues: Read and Write`
3. 如果使用 Fine-grained token，確認 token 的 expiration 未到期
4. 改用 Classic token 並選擇 `repo` scope 作為測試基準

---

## 關鍵重點
- MCP 解決的核心問題是讓 Claude 從被動接收資料轉變為主動查詢資料，消除手動橋接外部系統的需求
- `.mcp.json` 進入版本控制可讓團隊共享 MCP 設定，但敏感 token 必須透過環境變數注入，絕不寫死在配置檔中
- 多個 MCP server 同時啟用的 context 消耗不容忽視（5 個 server 約 55,000 tokens）；僅啟用當前任務需要的 server，並善用 Tool Search 自動模式
- 只需一次性查詢的場景直接用 `curl` 更簡單；MCP 的價值在於需要多步驟、動態互動的任務
- 只安裝來源可信的 MCP server；第三方 server 以你的身份操作外部系統，安裝前應審查原始碼

## 自我檢核
- 你目前的開發流程中，哪些步驟需要手動呼叫外部 API 再將結果貼入 Claude？這些是 MCP 的潛在使用場景。
- 同時啟用 GitHub、Slack、PostgreSQL、Figma、Playwright 五個 MCP server，但某次任務只需要查詢資料庫，你應該如何管理 context 消耗？
- 在高安全性的生產環境中，你會用什麼標準評估一個第三方 MCP server 是否值得信任？

---

⬅️ [上一章：Skills 系統](03-3-Skills系統.md) ｜ 📖 [回目錄](../00-索引與摘要.md) ｜ ➡️ [下一章：自訂 Subagents](../04-編排層/04-1-自訂Subagents.md)

🌐 [English Version](../../en/03-automation/03-4-mcp-servers.md)
