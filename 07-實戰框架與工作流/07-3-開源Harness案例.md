# 07-3 開源 Harness 案例
> 三個頂級開源專案的 Harness 設計，以及你可以直接偷的做法。

## 學習目標
- 理解為什麼分析別人的 Harness 是建立自己系統的有效捷徑
- 掌握三個開源案例各自的核心設計亮點和可移植元素
- 學會用七大概念框架對比不同 Harness 的設計選擇
- 能夠從多個案例提取最小可行組合，建立符合自己需求的 Harness

---

## 正文

### 1. 為什麼要看別人的 Harness

在建立自己的 Claude Code 工作流之前，先花時間研究已經被實戰驗證的開源 Harness，這是投資報酬率最高的學習方式。原因有三：

**第一，真實踩坑的結晶。** 每一個開源 Harness 的每一條規則，背後都是某次真實的錯誤或低效。這些設計決策都是付出時間成本換來的教訓，你可以免費站在這些踩坑經驗上。

**第二，設計模式的廣度。** 一個人在短時間內很難想出所有可能的架構選擇。看三個不同的 Harness，你會看到三種不同的問題視角，這讓你的設計思路更廣。

**第三，避免重新發明輪子的衝動。** 很多人在建立 Harness 時花大量時間在「設計架構」上，卻忽略了「根據自己的踩坑清單填入內容」才是讓 Harness 有效的關鍵。

但這裡有一個重要的警告：**直接複製別人的 Harness 是反模式。** 別人的規則反映的是他們的踩坑經驗、他們的技術棧、他們的工作習慣。如果你複製了 100 條規則，但其中 70 條和你的實際情況無關，你只是給自己的 context window 增加了噪音。

正確的做法是：**提取設計模式，用自己的踩坑內容填入。**

---

### 2. 案例一：claude-code-harness

**來源：** github.com/Chachamaru127/claude-code-harness

**核心設計：簡化入口 + 硬攔截**

這個專案解決了一個很常見的問題：隨著 Skill 數量增加，系統變得難以使用。它的做法是把 42 個舊 Skill 整合進 5 個動詞 Skill：

- `/setup`：初始化環境和依賴
- `/plan`：分析問題和設計方案
- `/work`：執行實作任務
- `/review`：審查程式碼品質
- `/release`：準備和執行發布

這 5 個動詞覆蓋了完整的開發週期，使用者不需要記住 42 個命令的語義，只需要知道「現在是哪個階段」。

**最值得關注的設計：9 條 TypeScript Runtime Guardrail（R01-R09）**

這是 claude-code-harness 最獨特的地方。多數 Harness 用 CLAUDE.md 的文字規則來約束 Claude 的行為，但文字規則有一個根本問題：Claude 可以理解規則，但在長對話中或複雜任務中，它有可能「忘記」或「繞過」文字規則。

TypeScript Guardrail 不是 prompt 警告，而是執行路徑上的程式碼攔截。當 Claude 試圖執行某個被禁止的操作時，Guardrail 在程式碼層面攔截，強制中止並提示正確做法。這從「提醒 Claude 遵守規則」升級到「讓違規在技術上不可能發生」。

9 條 Guardrail 的設計涵蓋常見的高風險操作：直接生產環境操作、跳過測試的合併、不安全的環境變數處理等。

**4 角度並行 Review**

`/review` 不是單一視角的審查，而是同時從四個角度進行：

1. **功能正確性**：程式碼是否實作了預期行為
2. **安全性**：是否存在 OWASP Top 10 等常見漏洞
3. **效能**：是否有明顯的效能問題（N+1 查詢、不必要的阻塞操作等）
4. **可維護性**：程式碼是否符合團隊規範、是否容易理解和修改

這四個角度並行進行意味著不會因為「功能看起來正常」就跳過安全審查，也不會因為「安全沒問題」就忽略效能。

**可以直接偷的做法：**
- TypeScript Guardrail 做硬攔截，把最重要的規則從文字層面提升到程式碼層面
- 4 角度 Review 結構，確保每次 review 都覆蓋功能、安全、效能、可維護性
- 把複雜系統的入口簡化為 5 個動詞，降低使用門檻

---

### 3. 案例二：everything-claude-code

**來源：** github.com/affaan-m/everything-claude-code

**核心設計：自動學習 + 規模化隔離**

這個專案的規模遠超一般個人 Harness：100+ Skill、28 個 Subagent 的效能優化系統。它解決的問題是：**如何讓 Harness 本身持續進化，而不是靠人工維護。**

**最值得關注的設計：learn → evolve 學習循環**

`/learn-eval` 是一個自動 pattern 提取工具。它分析近期的工作記錄、錯誤日誌和 review 意見，提取出重複出現的 pattern——無論是常見的錯誤模式、高效的解法模式，還是被反覆要求的程式碼結構。

`/evolve` 把 `/learn-eval` 提取出的 pattern 轉化成新的 Skill。這意味著 Harness 不是靜態的，而是從每次工作中學習並自動擴展能力。

這個循環的本質是：**把踩坑經驗自動提煉成可重用規則，不需要人工介入。**

**28 個 Subagent 隔離架構**

在大型專案中，不同語言、不同框架、不同模組的工作規則是不同的。如果所有規則都在同一個 context 中，它們會互相干擾。

28 個 Subagent 的設計是：每個 Subagent 只知道它需要知道的規則。一個 React Subagent 不會被 Python 的型別規則干擾，一個安全審計 Subagent 不會被 UI 開發規則分心。這種隔離讓每個 Subagent 在自己的領域內表現更專注、更可靠。

**按需安裝語言規則**

不同技術棧的規則不是預載的，而是在需要時才安裝。當你的任務切換到一個不同的語言或框架時，相關的規則集才被載入到 context 中。這避免了「所有規則同時在場」導致的 context 污染問題。

**可以直接偷的做法：**
- learn → evolve 學習循環，讓踩坑經驗自動提煉成規則
- Subagent 隔離架構，不同領域的規則不共享 context
- 按需安裝語言規則，避免不相關規則佔用 context 空間

---

### 4. 案例 3：OpenAI Codex 內部團隊

| 項目 | 說明 |
|------|------|
| 來源 | OpenAI 官方文章「Harness Engineering」（2026-02） |
| 規模 | 5 個月、約 100 萬行程式碼、約 1,500 個 PR、3→7 名工程師 |
| 約束 | 零手寫程式碼——每一行由 Codex（GPT-5）生成 |
| 核心原則 | Humans steer, Agents execute（人類掌舵，Agent 執行） |
| 速度 | 每人每天 3.5 個 PR，約 10 倍傳統速度 |

**設計亮點**：

1. **AGENTS.md 當地圖**：約 100 行，指向 `docs/` 目錄下的結構化知識庫（設計文件、執行計畫、產品規格、參考資料），實現漸進式揭露（progressive disclosure）

2. **嚴格的架構分層**：Types → Config → Repo → Service → Runtime → UI，依賴方向嚴格驗證，由自訂 linter 和結構測試機械式強制執行

3. **Agent-to-Agent Review**：Agent 自我 review → 請其他 Agent review → 迭代至滿意（「Ralph Wiggum Loop」），人類不必每次 review

4. **Garbage Collection Agent**：背景自動掃描偏差、開重構 PR，取代了初期每週五 20% 的手動清理時間

5. **可觀測性整合**：每個 worktree 有獨立的 log / metrics / trace 堆疊，Agent 可用 LogQL 和 PromQL 查詢，讓「確保啟動在 800ms 內」這類 prompt 變得可執行

**可偷的做法**：
1. AGENTS.md / CLAUDE.md 當地圖，`docs/` 存結構化知識——已在本課程 02-1 教過
2. 自訂 linter 錯誤訊息寫成 Agent 修復指令——低成本高回報（見 03-1 進階技巧）
3. Garbage Collection Agent 自動開重構 PR——取代人工審計

---

### 5. 七大概念 × 案例交叉對照表

理解四個案例的最有效方式，是用同一套概念框架對比它們的設計選擇：

| 概念 | claude-code-harness | everything-claude-code | gstack | OpenAI Codex |
|------|-------------------|----------------------|--------|-------------|
| 上下文架構 | Skills 漸進載入 | 按需安裝語言規則 | CLAUDE.md 當索引 | AGENTS.md 100 行 + docs/ |
| 架構約束 | 9 條 TS guardrail | 多語言 Rules 強制 | `/careful` 警告 | 6 層架構 + 自訂 linter |
| 自驗證循環 | `/harness-work all` | checkpoint evals | `/qa` 自動測試 | Agent-to-Agent review |
| 上下文隔離 | 3 Agent 分工 | 28 Subagent 防火牆 | 18 Skill 獨立 | 每 worktree 獨立環境 |
| 熵管理 | agent-trace.jsonl | `/learn-eval` → `/evolve` | `/document-release` | GC Agent 自動重構 |
| 可拆卸性 | v3 middleware 可移除 | 選擇性安裝 | Skill 檔案獨立 | 未明確討論 |
| Agent 可讀性 | — | — | — | linter 錯誤 = 修復指令 |

這個表格揭示了幾個有趣的設計差異：

**架構約束的強度不同。** claude-code-harness 選擇了最強的約束方式（程式碼層攔截），gstack 選擇了最輕的方式（警告提示）。這反映了不同的設計哲學：前者相信規則需要被強制執行，後者相信工程師應該能夠在必要時繞過規則。

**熵管理的自動化程度不同。** everything-claude-code 的 learn → evolve 循環是自動的，gstack 的 `/document-release` 是手動的。自動化的代價是複雜性，手動的代價是依賴人的紀律。

**可拆卸性的粒度不同。** everything-claude-code 可以選擇性安裝單個語言規則包，claude-code-harness 的 v3 middleware 可以整體移除，gstack 的每個 Skill 檔案都是獨立的。三者都保持了可拆卸性，但粒度不同。

---

## 教學舉例

### 場景：從零建立一個「最小 Harness 組合」

假設你是一個全端開發者，用 TypeScript + Next.js 獨立開發 SaaS 產品，每週處理 10 至 15 個功能和 bug fix 任務。你從來沒有建立過系統性的 Harness，每次都是臨時給 Claude 提示。你決定建立你的第一個 Harness。

從三個案例各偷一個最核心的設計，以下是建議的「最小 Harness 組合」：

**從 claude-code-harness 偷：5 個動詞入口**

這是最優先採用的，因為它解決了「我有 Harness 但記不住怎麼用」的問題。你的最小版本只需要三個 Skill：

- `/plan`：分析任務、列出步驟、識別風險
- `/work`：執行實作，帶入驗證條件
- `/review`：用 4 個角度審查（功能、安全、效能、可維護性）

你暫時不需要 `/setup` 和 `/release`，因為你的環境已經配置好，發布流程也已熟悉。先從最高頻使用的三個入口開始。

**從 everything-claude-code 偷：手動版學習循環**

你還沒有資源建立完整的自動 `/learn-eval` → `/evolve` 循環，但你可以建立手動版本：

在 CLAUDE.md 裡新增一個區塊叫做「本週踩坑」，每次任務完成後，花 5 分鐘回答：
1. 這次遇到了什麼 Claude 的錯誤或低效？
2. 下次怎麼在 prompt 或 Skill 中預防？

每週五，把「本週踩坑」區塊的內容提煉成新的 CLAUDE.md 規則或更新現有 Skill。這是手動的 learn → evolve，但它建立了相同的習慣：**每次踩坑都必須轉化成系統改進。**

**從 gstack 偷：/office-hours 逼問機制**

在你的 `/plan` Skill 裡加入一個強制步驟：在列出實作步驟之前，必須先回答三個問題（簡化版的 office-hours）：

1. 這個任務最可能失敗的地方在哪裡？
2. 有沒有比我第一直覺更簡單的解法？
3. 如果這個任務做錯了，修正的代價是什麼？

這三個問題比 gstack 的六個問題更輕量，但保留了最核心的功能：**強制在動手前把最重要的隱藏假設挖出來。**

**你的最小 Harness 結構**

```
CLAUDE.md              ← 核心規則 + 本週踩坑區塊
.claude/
  commands/
    plan.md            ← 3 個逼問問題 + 步驟分解
    work.md            ← 任務執行 + 驗證條件
    review.md          ← 4 角度審查清單
```

這個結構的總行數大約是 150 至 200 行，涵蓋最高頻的使用場景，不需要複雜的 Agent 架構。

**為什麼這個組合有效**

這三個設計彼此互補，沒有重疊：
- 5 個動詞入口解決「如何進入工作流」的問題
- 學習循環解決「如何讓系統持續改善」的問題
- 逼問機制解決「如何避免在錯誤方向上快速前進」的問題

更重要的是，這個組合的每一個元素都來自真實案例的驗證，而不是理論設計。你是在站在別人踩坑經驗的基礎上起步，而不是從零開始摸索。

**擴展路徑**

當你用這個最小組合工作了四到六週之後，你應該已經有了自己的踩坑清單。這時候，你可以根據具體的痛點決定下一步擴展：

- 如果你發現 Claude 在安全性上反覆犯同類型的錯誤，考慮引入 claude-code-harness 的 TypeScript Guardrail 概念，把規則從文字層面提升到程式碼層面
- 如果你的任務開始涉及多個技術棧，考慮引入 everything-claude-code 的按需載入規則包
- 如果你的任務規模擴大到需要多角色協作，考慮引入 gstack 的角色分工設計

這個路徑的核心邏輯是：**先用最小組合建立習慣，再根據真實遇到的摩擦決定擴展方向。** 不要在你還沒有踩到那個坑之前，就預防性地搬入你不需要的複雜性。

---

## 關鍵重點
- 分析開源 Harness 的正確姿勢是「提取設計模式，用自己的踩坑內容填入」，而不是直接複製
- claude-code-harness 最獨特的貢獻是 TypeScript Runtime Guardrail，把規則從文字層面提升到程式碼層面的硬攔截
- everything-claude-code 的 learn → evolve 循環解決了 Harness 需要人工維護的根本問題
- 七大概念框架（上下文架構、架構約束、自驗證循環、上下文隔離、熵管理、可拆卸性）可以用來系統性對比任何兩個 Harness 的設計差異
- 最小 Harness 組合原則：從最高頻使用場景出發，用實際踩坑驅動擴展，不預防性引入複雜性

---

## 自我檢核
- 用七大概念框架審視你目前的工作流（或 CLAUDE.md），哪個概念是你目前完全缺失的？這個缺失有沒有對你造成實際痛點？
- 如果你今天要建立一個最小 Harness，你的「本週踩坑」清單裡最重要的三條規則是什麼？
- TypeScript Runtime Guardrail 和 CLAUDE.md 文字規則的核心區別是什麼？在你目前的工作中，有哪條規則值得升級成硬攔截？
