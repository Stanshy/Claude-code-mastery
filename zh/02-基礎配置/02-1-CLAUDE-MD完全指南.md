# CLAUDE.md 完全指南
> CLAUDE.md 是 Claude Code 最重要的配置檔案，它決定了 Claude 從第一秒就理解你的專案規則，而不是每次從零猜測。

## 學習目標
- 理解 CLAUDE.md 為何是所有配置中優先順序最高的檔案
- 掌握四種檔案位置的適用場景與作用範圍
- 學會判斷哪些內容應該寫入、哪些應該排除
- 能在 10 分鐘內為真實專案建立有效的 CLAUDE.md

---

## 正文

### 1. 為什麼 CLAUDE.md 是最重要的配置

CLAUDE.md 是 Claude Code 在每次會話開始時**自動載入**的指令檔案。它決定了 Claude 對你的專案的理解方式。沒有 CLAUDE.md，Claude 每次都從零開始猜測你的偏好；有了 CLAUDE.md，Claude 從第一秒就知道你的規則。

這不是一般的說明文件。它是寫給 Claude 看的操作手冊，直接影響 Claude 的每一個決策：用哪個指令執行測試、遇到錯誤時怎麼處理、程式碼風格如何取捨。

Claude Code 的創建者 Boris Cherny 說：「每次 Claude 做錯了什麼，就把它加進 CLAUDE.md，讓它永遠不再重蹈覆轍。」他本人的 CLAUDE.md 僅約 2,500 tokens，由整個團隊在 code review 過程中持續迭代。這句話揭示了 CLAUDE.md 的本質：它是一份**活的文件**，從錯誤中成長，不是一次性寫完的靜態說明。

### 2. 檔案位置與作用範圍

CLAUDE.md 可以存在於多個位置，每個位置的作用範圍不同：

| 位置 | 範圍 | 用途 |
|------|------|------|
| `./CLAUDE.md`（專案根目錄） | 單一專案 | commit 到 git，團隊共享 |
| `~/.claude/CLAUDE.md` | 所有專案 | 個人偏好（如語言、風格） |
| 子目錄 `packages/api/CLAUDE.md` | 特定子模組 | monorepo 中各子系統獨立規則 |
| `/etc/claude-code/CLAUDE.md` | 組織級別 | 管理員統一部署 |

Claude 會自動載入從工作目錄到根目錄路徑上的所有 CLAUDE.md，以及 `~/.claude/CLAUDE.md`。多個 CLAUDE.md **合併生效**，不會互相覆蓋。

實際場景舉例：你在 `packages/api/` 目錄下工作時，Claude 會同時載入 `./CLAUDE.md`（根目錄規則）、`packages/api/CLAUDE.md`（API 模組專屬規則）、`~/.claude/CLAUDE.md`（你的個人偏好）。三層規則同時生效，讓你不需要在每個子目錄重複全局設定。

### 3. 應包含 vs 應排除

這是初學者最常犯錯的地方。CLAUDE.md 的核心原則是：**只寫 Claude 猜不到的內容**。

| 應包含 | 應排除 |
|--------|--------|
| Claude 猜不到的 bash 指令（如 `pnpm run dev:local`） | Claude 能從 package.json 推斷的內容 |
| 與預設不同的程式碼風格（如 tab vs space） | 標準語言慣例（如 Python 的 PEP8） |
| 測試指令和偏好（如 `jest --watch`） | 詳細 API 文件（改為 `@` 匯入） |
| 專案特有的架構決策 | 經常變動的資訊 |
| 必須遵守的規範（用 `IMPORTANT` 或 `YOU MUST` 強調） | 顯而易見的做法 |

寫「這是一個 Node.js 專案」是浪費 context。Claude 看到 `package.json` 自然知道。但寫「使用 ES module 語法，不使用 CommonJS」就有價值，因為 Node.js 兩種語法都支援，Claude 沒有額外線索時無從判斷你的選擇。

強制性規則要用明確語氣。「最好使用」和「YOU MUST 使用」對 Claude 的行為影響截然不同。對於不可妥協的規則，用 `IMPORTANT:` 或 `YOU MUST` 標記。

### 4. `/init` 快速起步

在專案目錄執行 `/init`，Claude 會自動分析 codebase 並生成初始版本的 CLAUDE.md。這是最快的起步方式，5 分鐘內就有一份可用的草稿。

但要注意：`/init` 產出的版本通常**需要人工精煉**。自動生成的內容傾向於把所有能描述的東西都寫進去，包括很多 Claude 其實能自己推斷的內容。生成後的第一件事是刪除冗餘，保留只有你才知道的規則。

### 5. `@` 匯入語法

CLAUDE.md 支援 `@` 引用外部檔案，避免把所有內容塞進單一檔案造成過載：

```markdown
@README.md
@docs/api-spec.md
@.claude/rules/testing.md
```

這讓你可以把 CLAUDE.md 本身控制在簡潔的範圍內，同時按需引入更深層的說明。API 規格文件可能有幾百行，不適合直接放進 CLAUDE.md，但用 `@` 引入後 Claude 在需要時仍然能參考到。

### 6. 大小控制策略

CLAUDE.md 越長，Claude 忽略部分規則的機率越高。這不是 Claude 的缺陷，而是 context window 的工程現實。

幾種主流策略：

- **Boris Cherny 方法**：維持約 2,500 tokens（約 200 行），全團隊共同維護，在每次 code review 中迭代
- **精簡學派**：控制在 60 行以內，只放普遍適用的規則
- **David Chu 整合觀點**：先用 Boris 方法培養「把錯誤加進 CLAUDE.md」的習慣，等規則成熟後，把頻繁觸發的規則「畢業」到 Hook / Skill / Agent，讓 CLAUDE.md 回到精簡狀態

實務建議：**CLAUDE.md 不超過 200 行**。超過這個數字就要考慮分拆到 `.claude/rules/` 目錄，或透過 `@` 引用，或轉化為自動化的 Hook。

### 8. 為 Agent 可讀性設計（而非人類）

OpenAI 在使用 Codex 的實踐中發現一個反直覺的原則：codebase 的文件和結構應該首先為 Agent 的可讀性最佳化。

具體做法：
- **所有決策留在 repo 內**：架構決策、設計文件、產品規格全部存在 `docs/` 目錄，而非外部工具（Google Docs、Notion、Slack）。從 Agent 的角度看，不在 repo 中的知識等同於不存在。
- **偏好「無聊」的技術**：選擇 composable、API 穩定、在訓練資料中充分表示的技術。Agent 對主流技術的理解比冷門框架更可靠。某些情況下，讓 Agent 重新實作功能子集，比處理不透明的第三方套件更便宜。
- **自訂 linter 錯誤訊息**：linter 的錯誤描述應該寫成 Agent 可直接執行的修復指令，而非只描述問題。

這個觀點對 CLAUDE.md 的啟示是：CLAUDE.md 不只是給 Claude 看的規則，它是 Agent 理解整個專案的入口。任何 Agent 無法從 repo 中發現的知識，等同於不存在。

### 9. 記憶系統與 CLAUDE.md 的分工

Claude Code 有兩套記憶機制，混淆兩者是常見的誤解：

| 系統 | CLAUDE.md | Auto Memory |
|------|-----------|-------------|
| 誰寫 | 你 | Claude |
| 內容 | 規則和指令 | 學習和模式 |
| 何時載入 | 每次會話 | 每次會話 |
| 用途 | 永遠適用的規則 | 專案特定洞察 |

Auto Memory 位於 `~/.claude/projects/<project>/memory/`，Claude 在工作過程中自動寫入。例如 Claude 發現你的測試檔案命名慣例後，會記錄下來供後續會話參考。

你不需要手動管理 Auto Memory。你的職責是維護 CLAUDE.md 中的規則；Claude 的職責是在 Auto Memory 中積累觀察。兩套系統互補，不要把本該放在 CLAUDE.md 的規則依賴 Auto Memory 來記憶，因為 Auto Memory 的內容你無法直接控制。

---

## 實作範例：為 AI Dev Assistant 撰寫 CLAUDE.md
> 主案例延續：在 AI Dev Assistant 專案中，這是「架構能力線」的第一步，也是整個 AI Dev Assistant 專案後續所有操作的基礎。

**目標**：為 ai-dev-assistant 專案撰寫一份有效的 CLAUDE.md，確保後續所有 Claude 會話自動遵循專案規則

**前置條件**：已完成 01-2（專案骨架建立），`ai-dev-assistant/` 目錄已存在並包含 `package.json`

**步驟**：

1. 進入專案目錄並啟動 Claude Code：
   ```bash
   cd ai-dev-assistant
   claude
   ```

2. 在 Claude Code 中執行 `/init`，讓 Claude 自動分析專案結構並生成初版 CLAUDE.md：
   ```
   /init
   ```

3. Claude 會分析 `package.json`、`tsconfig.json`、目錄結構等，生成草稿。等待完成後，用編輯器開啟 `CLAUDE.md`。

4. 刪除草稿中所有 Claude 能自行推斷的內容，根據以下範本精煉（這份範本是 ai-dev-assistant 專案的正式版本）：

   ```markdown
   # AI Dev Assistant

   CLI 工具，自動執行 code review、修 bug、分析 PR、生成測試。

   ## 技術棧
   - Node.js 18.x (TypeScript 5.x)
   - 測試：Jest 29.x
   - Lint：ESLint 8.x + Prettier 3.x

   ## 核心規則
   - 使用 ES module 語法（import/export），不使用 CommonJS（require）
   - 所有函數必須有明確的 TypeScript 型別標註
   - commit 前必須通過 `npm test` 和 `npm run lint`
   - 錯誤處理使用自訂 Error class，不使用 string throw

   ## 常用指令
   - 建構：`npm run build`
   - 測試：`npm test`
   - 單一測試：`npx jest --testPathPattern=<filename>`
   - Lint：`npm run lint`
   - 格式化：`npx prettier --write .`

   ## 專案結構
   - `src/` — 核心邏輯
   - `tests/` — Jest 測試
   - `scripts/` — 自動化腳本
   - `.claude/` — Claude Code 設定
   ```

5. 儲存 `CLAUDE.md` 後，在 Claude Code 中清除當前會話並測試規則是否生效：
   ```
   /clear
   ```
   然後輸入一個簡單任務，例如：「在 src/ 中建立一個 utils.ts 並加入一個 formatDate 函數」。觀察 Claude 是否使用 ES module 語法並加上 TypeScript 型別標註。

6. 如果 Claude 違反了某條規則（例如使用了 `require` 而非 `import`），立即把這個違規情境加進 CLAUDE.md 的核心規則中，並說明**為什麼**這條規則存在：
   ```markdown
   ## 核心規則
   - IMPORTANT: 必須使用 ES module 語法（import/export）。
     此專案的 package.json 設定 "type": "module"，使用 require 會導致執行期錯誤。
   ```

**預期結果**：
- 專案根目錄出現 `CLAUDE.md`，檔案不超過 200 行
- 後續所有 Claude 會話自動遵循技術棧規則（ES module、TypeScript 型別標註）
- Claude 執行測試時使用 `npm test` 而非猜測其他指令
- `CLAUDE.md` 已 commit 到 git，團隊成員 clone 後立即生效

**常見錯誤**：
- **CLAUDE.md 寫太長（超過 300 行）**：Claude 開始忽略後段規則，越重要的規則越容易被遺漏。解法：精簡到 200 行以內，把深層說明用 `@` 引用或移到 `.claude/rules/` 子目錄。
- **寫了 Claude 能自己推斷的內容**（例如「這是一個 Node.js 專案」、「使用 TypeScript」）：浪費有限的 context 空間。解法：只寫 Claude 在沒有額外提示時無法正確猜到的規則。
- **把規則寫成建議語氣**（例如「最好使用 ES module」）：Claude 會把它當作可選建議而非硬性要求。解法：強制規則一律用 `IMPORTANT:` 或 `YOU MUST` 前綴。

---

## 關鍵重點
- CLAUDE.md 在每次會話開始時自動載入，是影響 Claude 行為最直接的機制
- 只寫 Claude 猜不到的內容，避免浪費 context；強制規則用 `IMPORTANT` 或 `YOU MUST` 標記
- 四個位置的 CLAUDE.md 合併生效：根目錄（團隊共享）、子目錄（子模組專屬）、`~/.claude/`（個人偏好）、`/etc/claude-code/`（組織部署）
- 用 `/init` 快速生成草稿，再人工精煉刪除冗餘內容
- CLAUDE.md 控制在 200 行以內；超過就分拆或轉化為 Hook

## 自我檢核
- 你的 CLAUDE.md 中有沒有 Claude 能從 `package.json` 或目錄結構自行推斷的內容？如果有，刪除它。
- 當 Claude 在你的專案中做了一件錯誤的事，你是否把這個錯誤和正確做法加進了 CLAUDE.md？
- 你的 CLAUDE.md 現在有幾行？如果超過 200 行，哪些內容可以移到 `@` 引用的獨立檔案？

---

⬅️ [上一章：安裝與第一次成功](../01-認識Claude-Code/01-2-安裝與第一次成功.md) ｜ 📖 [回目錄](../00-索引與摘要.md) ｜ ➡️ [下一章：設定檔與權限系統](02-2-設定檔與權限系統.md)

🌐 [English Version](../../en/02-config/02-1-claude-md-guide.md)
