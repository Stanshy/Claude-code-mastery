# 05-1 Agent Teams
> 讓多個 Claude session 協作，突破單一 session 的能力邊界。

## 學習目標
- 理解 Agent Teams 的協作架構與適用場景
- 能夠啟用並配置 Agent Teams 實驗性功能
- 學會設計 Lead 與 Teammate 的任務分配策略
- 判斷何時應該使用 Teams，何時用 Subagent 即已足夠

## 正文

### 1. 為什麼需要 Agent Teams

每個 Claude session 都有 context 上限與能力邊界。當任務規模夠大，單一 session 需要記住「正在寫的功能」同時還要「分析測試覆蓋率」，這兩件事相互競爭 context，導致任一方的品質下降。

Agent Teams 的設計哲學是分工而非堆疊：一個 Lead session 負責協調全局，多個 Teammate sessions 各自在共享的 codebase 上獨立執行子任務，完成後回報結果。Team 成員可以直接互相通訊，不需要全部通過 Lead 轉發。這讓大型任務得以真正平行推進，而非偽裝的平行。

Agent Teams 目前是實驗性功能，代表 API 和行為可能隨版本調整。在生產系統中使用前，請釘定 Claude Code 版本並做充分測試。

### 2. 架構

Agent Teams 的三個角色：

**Team Lead**：整體任務的指揮者。負責解析使用者需求、拆分子任務、分配給適合的 Teammate、監控進度、最終整合結果。Lead 不直接修改程式碼，它的工作是協調。

**Teammates**：執行者。每個 Teammate 接受一個邊界清晰的子任務，在自己的工作範圍內獨立決策。Teammates 之間可以直接通訊，例如「負責功能的 Teammate 告知負責測試的 Teammate 函數簽名已確定」。

**啟用方式**：設定環境變數，無需修改任何程式碼：

```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

啟用後，在 Claude Code session 中描述需要多人協作的任務，Lead 會自動判斷是否派生 Teammates。

### 3. 適用場景

Agent Teams 的價值在於「真正獨立的平行工作」。以下場景符合這個條件：

**研究與審查**：一個 Teammate 深入研究某個問題（例如分析現有的認證實作），另一個同步 review 相關的 security 規範，兩個工作流互不干擾，最後交給 Lead 綜合。

**新模組開發**：功能實作與測試撰寫可以拆分。負責實作的 Teammate 完成函數簽名後通知測試 Teammate，後者立即開始撰寫測試，而前者繼續完善實作細節。

**多假設除錯**：當一個 bug 有三個可能原因，傳統做法是依序驗證。Agent Teams 可以讓三個 Teammates 同時驗證三個假設，Lead 根據結果選出正確的修復方案。

**跨層協作**：前端 Teammate 修改 API 呼叫邏輯，後端 Teammate 同步調整對應的 endpoint，兩者通訊確保介面一致，而非等一方完成後另一方才能開始。

### 4. Agent-to-Agent Review

OpenAI 在實踐中發展出一個更激進的模式：Agent 之間互相 review PR，人類不必每次介入。

流程（OpenAI 稱為「Ralph Wiggum Loop」）：

1. Agent A 完成實作，開 PR
2. Agent A 自行 review 自己的 PR
3. Agent B（本地或雲端）對 PR 進行獨立 review
4. Agent A 回應 review 回饋並修改
5. 循環迭代直到所有 reviewer Agent 滿意
6. 人類可以 review，但不是必須的

這個模式在高吞吐量環境中有意義：當每人每天產出 3.5 個 PR 時，人類 review 成為瓶頸。Agent-to-Agent review 解放了人類的注意力，讓人類專注在方向判斷而非程式碼細節。

目前限制：需要高度成熟的 Harness（架構約束 + 自動化測試 + 結構化 linter）作為品質保障的底層，否則 Agent review 的品質不可靠。在你的 Harness 尚未建立足夠的架構約束之前，人類 review 仍然是必要的。

---

### 5. 限制與注意事項

**實驗性功能的風險**：`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 中的 `EXPERIMENTAL` 不是裝飾字。功能本身可能有未知邊界行為，在重要專案使用前，先在測試分支驗證。

**計算預算**：每個 Teammate 都消耗獨立的 token。三個 Teammates 同時運行的成本約是單一 session 的三倍。在任務拆分時，要確保平行帶來的時間節省值得這個成本。

**規模建議**：Team 規模以 3-5 個 agent 為宜。超過五個 agent 後，Lead 的協調負擔反而成為瓶頸——它需要處理更多的進度回報、衝突解決、結果整合，效率反而不如較小的 Team。

**任務邊界設計**：Teammates 最常見的問題是修改了相同的檔案，導致合併衝突。設計任務時，檔案級別的隔離是最可靠的策略：Teammate A 負責 `src/analyzer.ts`，Teammate B 負責 `tests/analyzer.test.ts`，兩者不重疊。

**務實的選擇原則**：簡單任務（單一檔案、單一目標）用 Subagent 即可。只有當任務確實需要「多個人同時獨立工作才能提速」時，才值得引入 Agent Teams 的協調成本。不要為了用新功能而用新功能。

## 實作範例：3 人 Agent Team 開發新模組
> 主案例延續：在 AI Dev Assistant 專案中，用 Agent Team 平行開發 PR 分析模組，一個 Teammate 實作功能，一個 Teammate 撰寫測試。

**目標**：用 Agent Team 為 ai-dev-assistant 建立 PR 分析模組，包含實作與測試，並透過 Team 機制完成協作。

**前置條件**：
- 已完成 04-3（Worktree 設定），專案根目錄為 `C:\projects\ai-dev-assistant`
- Node.js 18.x、TypeScript 5.x、Jest 29.x 環境就緒
- Claude Code 已安裝並可正常執行

**步驟**：

1. 設定實驗性功能環境變數（此步驟每次新開終端都需要執行）：
   ```bash
   export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
   echo $CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS  # 確認輸出為 1
   ```

2. 切換到專案目錄並啟動 Lead session：
   ```bash
   cd /c/projects/ai-dev-assistant
   claude
   ```

3. 在 Lead session 中輸入以下任務描述（完整貼入，讓 Lead 能正確拆分）：
   ```
   我需要為 ai-dev-assistant 建立 PR 分析模組。
   請拆分為以下兩個子任務並分配給 Teammates：
   - Teammate A：在 src/analyzer.ts 中實作 analyzePR 函數，
     接受 PR diff 字串，回傳包含 issues 和 suggestions 陣列的物件。
   - Teammate B：在 tests/analyzer.test.ts 中撰寫 analyzePR 的測試，
     包含正常輸入和空字串的邊界測試。
   Teammate A 完成函數簽名後，請通知 Teammate B。
   ```

4. 觀察 Lead 的任務分配輸出。Lead 會顯示它正在派生的 Teammates 及各自的任務範圍。正常情況下，你會看到類似「Spawning Teammate for src/analyzer.ts implementation」的訊息。

5. 觀察兩個 Teammates 獨立工作的過程。Teammate A 建立 `src/analyzer.ts` 並定義函數簽名後，Lead 介面會顯示「Notifying Teammate B of function signature」。

6. 待兩個 Teammates 都完成後，Lead 會整合結果並回報。確認檔案存在：
   ```bash
   ls /c/projects/ai-dev-assistant/src/analyzer.ts
   ls /c/projects/ai-dev-assistant/tests/analyzer.test.ts
   ```

7. 執行測試確認整合結果正確：
   ```bash
   cd /c/projects/ai-dev-assistant
   npx jest tests/analyzer.test.ts
   ```

8. 如果測試失敗，回到 Lead session 請它協調修復：
   ```
   測試失敗，錯誤訊息如下：[貼上錯誤]。請協調 Teammates 修復。
   ```

**預期結果**：
- Lead session 顯示清楚的任務分配記錄和兩個 Teammates 的進度
- `src/analyzer.ts` 包含 `analyzePR` 函數，TypeScript 型別完整
- `tests/analyzer.test.ts` 包含至少三個測試案例（正常流程、空字串、格式錯誤）
- `npx jest tests/analyzer.test.ts` 全數通過

**常見錯誤**：
- **環境變數未設定，Claude 啟動了一般模式而非 Team 模式**：症狀是 Claude 直接開始獨立完成任務，沒有顯示「Spawning Teammate」。解法：退出 session，執行 `export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`，再重新啟動 `claude`。注意 export 只在當前 shell 有效，新開終端要重新執行。
- **Teammates 同時修改了相同檔案導致內容衝突**：症狀是某個檔案內容異常，或 TypeScript 編譯報出不應存在的錯誤。根本原因是任務描述中沒有明確劃分檔案責任。解法：重新描述任務時，明確指定「Teammate A 只修改 src/analyzer.ts，Teammate B 只修改 tests/analyzer.test.ts，不觸碰對方的檔案」。

## 關鍵重點
- Agent Teams 解決的是「任務太大、需要真正平行」的問題，不是「任務太難」的問題
- Lead 負責協調，Teammates 負責執行，分工清晰是 Team 正常運作的前提
- 啟用 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 後，在任務描述中主動要求 Lead 拆分，它才會派生 Teammates
- 3-5 個 Teammates 是效率最佳的規模，超過後協調成本上升快於效益
- 任務設計時以檔案為邊界隔離工作範圍，是避免 Teammates 衝突最可靠的方法

## 自我檢核
- 你能說出 Lead 和 Teammate 各自負責的職責，以及兩者如何通訊嗎？
- 給定一個任務（例如「為 ai-dev-assistant 新增 user authentication 模組」），你能設計一個合理的 3-Teammate 分工方案，並說明每個 Teammate 的檔案範圍嗎？
- 什麼條件下你會選擇 Subagent 而非 Agent Teams？

---

⬅️ [上一章：平行開發與 Worktree](../04-編排層/04-3-平行開發與Worktree.md) ｜ 📖 [回目錄](../00-索引與摘要.md) ｜ ➡️ [下一章：Headless 模式與 CI/CD 整合](05-2-Headless模式與CI-CD整合.md)

🌐 [English Version](../../en/05-autonomous/05-1-agent-teams.md)
