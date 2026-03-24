# 05-3 Agent SDK
> 用程式碼控制 Claude Code，將 AI 能力嵌入你自己的應用與自動化工具。

## 學習目標
- 理解 Agent SDK 與 CLI 的根本差異，以及選擇各自的判斷依據
- 能夠安裝並使用 TypeScript SDK 的 `query` API 執行任務
- 掌握 `allowedTools` 和 `maxTurns` 兩個核心參數的作用
- 為 ai-dev-assistant 建立可在任意目錄執行的 `assistant` CLI 指令

## 正文

### 1. 為什麼需要 Agent SDK

CLI 是設計給開發者直接坐在終端機前使用的工具。它的互動介面、即時輸出、確認提示，都預設「有人在看著螢幕」。當你的需求從「我要用 Claude Code」變成「我要讓我的程式呼叫 Claude Code」，CLI 就不再是正確的工具。

Agent SDK 提供程式化介面，讓 Claude Code 的能力成為你程式碼中的一個函數呼叫。這解鎖了幾個 CLI 做不到的場景：

**嵌入現有應用**：你的 Node.js 後端在使用者提交 bug report 時，自動呼叫 Claude Code 分析相關程式碼並生成修復建議，直接回傳給使用者，不需要人工介入。

**自定義 CLI 工具**：你的團隊有特定的工作流程，需要一個指令把「分析 PR」、「生成 changelog」、「更新文件」串在一起，一個指令全部搞定。

**CI pipeline 整合**：在 CI 腳本中，根據測試失敗的類型動態決定讓 Claude Code 執行哪種修復策略，並把結果以結構化物件傳給下一個 pipeline 步驟。

SDK 提供 Python 和 TypeScript 兩種語言。本章以 TypeScript 為主，因為 ai-dev-assistant 本身是 TypeScript 專案，可以直接整合。

### 2. SDK vs CLI

理解兩者的差異，才能在對的場景選對工具：

| 面向 | CLI | SDK |
|------|-----|-----|
| 使用方式 | 終端互動，人工輸入 | 程式碼呼叫，自動執行 |
| 適合的使用者 | 開發者日常工作 | 工具開發者、自動化工程師 |
| 輸出格式 | 格式化文字，為人類可讀設計 | 結構化物件，為程式處理設計 |
| 會話管理 | 手動操作（`--session-id`） | 程式控制，可以根據邏輯決定 |
| 錯誤處理 | 顯示在終端，人工判斷 | try/catch，程式化處理 |
| 整合難度 | 需要 shell 呼叫或 child_process | 直接 import，原生整合 |

關鍵判斷問題：「這個任務是我自己要執行，還是我要讓程式幫我執行？」前者用 CLI，後者用 SDK。

### 3. 安裝

SDK 套件名稱和語言分別對應：

```bash
# TypeScript / Node.js 專案
npm install @anthropic-ai/claude-agent-sdk

# Python 專案
pip install claude-agent-sdk
```

安裝前確認環境變數已設定，SDK 使用相同的 API key：

```bash
echo $ANTHROPIC_API_KEY  # 必須有值，否則 SDK 呼叫會直接拋錯
```

### 4. 核心 API

TypeScript SDK 的主要入口是 `query` 函數。它接受一個 prompt 和 options，回傳一個 async iterator，你可以用 `for await...of` 逐步處理 Claude 的回應：

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Find and fix the bug in src/fixer.ts",
  options: {
    allowedTools: ["Read", "Edit", "Bash", "Grep", "Glob"],
    maxTurns: 20,
  },
})) {
  if (message.type === "text") {
    console.log(message.content);
  }
}
```

Python 版本的對應寫法：

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    async for message in query(
        prompt="Find and fix the bug in src/fixer.ts",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Edit", "Bash", "Grep", "Glob"],
            max_turns=20,
        ),
    ):
        if message.type == "text":
            print(message.content)

asyncio.run(main())
```

兩個最重要的 options 參數：

**`allowedTools`**：明確列出 Claude 可以使用的工具。不列出就代表禁用。這是安全邊界的第一道防線。對於只需要讀寫程式碼的任務，使用 `["Read", "Edit", "Bash", "Grep", "Glob"]` 已足夠。如果任務涉及網路查詢，加入 `"WebSearch"` 和 `"WebFetch"`。

**`maxTurns`**：Claude 在一次 `query` 調用中最多可以執行幾輪工具調用。一輪代表一次「Claude 思考 → 呼叫工具 → 取得結果」的循環。複雜任務（分析多個檔案、多步修復）需要更多 turns。建議從 20 開始，觀察實際消耗後再調整。設太低會導致任務中途截斷。

### 5. 內建工具

SDK 提供與 CLI 完全相同的工具集，在 `allowedTools` 中用字串名稱引用：

| 工具名稱 | 功能 | 常見用途 |
|----------|------|----------|
| `Read` | 讀取檔案內容 | 分析程式碼 |
| `Write` | 建立新檔案 | 生成新模組 |
| `Edit` | 修改現有檔案 | 修復 bug、重構 |
| `Bash` | 執行 shell 指令 | 跑測試、build |
| `Glob` | 檔案模式匹配 | 尋找特定類型的檔案 |
| `Grep` | 內容搜尋 | 找出所有使用某函數的地方 |
| `WebSearch` | 搜尋網路 | 查詢最新文件或 CVE |
| `WebFetch` | 抓取網頁內容 | 取得特定 URL 的文件 |
| `AskUserQuestion` | 詢問使用者 | 需要人類確認的決策點 |

原則：只給任務實際需要的工具。`allowedTools` 越窄，行為越可預測，出現非預期副作用的機率越低。

## 實作範例：用 SDK 建立 AI Dev Assistant CLI 入口
> 主案例延續：在 AI Dev Assistant 專案中，用 TypeScript SDK 建立 `assistant` CLI 指令，讓使用者可以在任意目錄執行 `assistant "修復 login bug"` 直接呼叫 Claude Code 執行任務。

**目標**：為 ai-dev-assistant 建立全域可用的 `assistant` CLI 工具。使用者輸入一段任務描述，Claude 自主完成並在終端輸出結果，不需要進入 Claude Code 互動介面。

**前置條件**：
- 已完成 04-1（Subagents 設定），ai-dev-assistant 專案結構完整
- Node.js 18.x、TypeScript 5.x、`ts-node` 或 `tsc` 可用
- `ANTHROPIC_API_KEY` 環境變數已設定
- `package.json` 中已有 `build` 腳本（`tsc`）

**步驟**：

1. 安裝 Agent SDK：
   ```bash
   cd /c/projects/ai-dev-assistant
   npm install @anthropic-ai/claude-agent-sdk
   ```
   安裝完成後確認 `node_modules/@anthropic-ai/claude-agent-sdk` 目錄存在。

2. 在 `src/index.ts` 中建立 CLI 入口邏輯（若檔案已存在，替換為以下內容；若不存在則新建）：
   ```typescript
   #!/usr/bin/env node
   import { query } from "@anthropic-ai/claude-agent-sdk";

   const task = process.argv.slice(2).join(" ");

   if (!task) {
     console.error("Usage: assistant <task>");
     console.error('Example: assistant "Fix the login bug in src/auth.ts"');
     process.exit(1);
   }

   async function main(): Promise<void> {
     console.log(`Running: ${task}\n`);

     for await (const message of query({
       prompt: task,
       options: {
         allowedTools: ["Read", "Edit", "Bash", "Grep", "Glob"],
         maxTurns: 30,
       },
     })) {
       if (message.type === "text") {
         process.stdout.write(message.content);
       }
     }

     console.log("\nDone.");
   }

   main().catch((err) => {
     console.error("Error:", err.message);
     process.exit(1);
   });
   ```

3. 在 `package.json` 中加入 `bin` 欄位，讓 npm 知道這個套件提供一個可執行指令：
   ```json
   {
     "bin": {
       "assistant": "./dist/index.js"
     }
   }
   ```
   注意：`bin` 指向的是編譯後的 `dist/index.js`，不是 `src/index.ts`。

4. 確認 `tsconfig.json` 的 `outDir` 設定為 `"./dist"`，且 `target` 為 `ES2020` 或更高（以支援 `for await...of`）：
   ```json
   {
     "compilerOptions": {
       "target": "ES2020",
       "module": "commonjs",
       "outDir": "./dist",
       "strict": true
     }
   }
   ```

5. 編譯 TypeScript：
   ```bash
   npm run build
   ```
   編譯成功後，`dist/index.js` 應該存在。

6. 為編譯後的檔案加上執行權限（Linux/macOS 必要，Windows 可跳過此步驟但不影響 `npm link`）：
   ```bash
   chmod +x dist/index.js
   ```

7. 用 `npm link` 將這個套件連結到全域，讓 `assistant` 指令在系統任何位置都可執行：
   ```bash
   npm link
   ```
   `npm link` 在全域 node_modules 建立符號連結，指向目前的專案目錄。

8. 在任意目錄測試指令是否正常運作：
   ```bash
   cd /tmp
   assistant "列出 ai-dev-assistant 的 src/ 目錄下所有 TypeScript 檔案，並說明每個檔案的用途"
   ```
   Claude 會執行 Glob 工具找到檔案，然後用 Read 工具讀取每個檔案，最後輸出說明。

**預期結果**：
- `assistant "任何任務描述"` 可以在系統任何目錄執行
- 終端即時顯示 Claude 的思考過程和執行結果（stream 輸出）
- 任務完成後顯示 `Done.` 並正常退出（exit code 0）
- 若任務描述為空，顯示 Usage 提示並以 exit code 1 退出

**常見錯誤**：
- **`Cannot find module '@anthropic-ai/claude-agent-sdk'` 錯誤**：SDK 未安裝，或 `npm install` 在錯誤的目錄執行。解法：確認你在 `/c/projects/ai-dev-assistant` 目錄下執行 `npm install @anthropic-ai/claude-agent-sdk`，並確認 `package.json` 的 `dependencies` 中出現這個套件名稱。
- **執行 `assistant` 時出現 `Permission denied: dist/index.js`（Linux/macOS）**：`dist/index.js` 缺少可執行位元，且第一行缺少 shebang。解法：確認 `src/index.ts` 第一行是 `#!/usr/bin/env node`（注意這必須在編譯前就存在），重新執行 `npm run build`，然後執行 `chmod +x dist/index.js`。
- **`npm link` 後 `assistant` 指令找不到（command not found）**：npm 全域 bin 目錄不在 PATH 中。執行 `npm bin -g` 取得全域 bin 目錄路徑，並將該路徑加入 `~/.bashrc` 或 `~/.zshrc` 的 `PATH` 變數中，重新載入 shell 後再試。

## 關鍵重點
- Agent SDK 的核心價值是「讓程式碼呼叫 Claude Code」，而非取代 CLI 的互動使用場景
- `query` 回傳 async iterator，`for await...of` 處理串流輸出是最自然的消費方式
- `allowedTools` 是安全邊界，只授予任務實際需要的工具，過寬的權限增加非預期行為的風險
- `maxTurns` 控制任務深度，太低導致複雜任務截斷，建議從 20 開始觀察後調整
- `npm link` 是本地開發 CLI 工具的標準方式，發布到 npm 後使用者改用 `npm install -g`

## 自我檢核
- 你能說出 `allowedTools: ["Read", "Grep", "Glob"]` 和 `allowedTools: ["Read", "Edit", "Bash", "Grep", "Glob"]` 各自適合什麼類型的任務，以及為什麼要區分嗎？
- 如果 `assistant "修復 src/auth.ts 的 bug"` 執行到一半就停止輸出並顯示截斷訊息，你會調整哪個參數，調整為多少是合理的起點？
- 從 SDK 的角度，`query` 函數的 async iterator 設計（而非一次回傳完整結果）解決了什麼實際問題？
