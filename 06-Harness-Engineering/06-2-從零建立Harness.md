# 06-2 從零建立 Harness
> 先讓 Agent 裸跑，觀察它犯什麼錯，再針對性加約束——每一條進入 Harness 的規則，都應該對應一個已發生的真實問題。

## 學習目標
- 理解「踩坑驅動」的 Harness 建設哲學，以及為什麼不應該預防性設計
- 掌握從裸跑到完整 Harness 的六步流程，包含每步的具體產物
- 辨識並避免三種常見的 Harness 建設反模式
- 完成 AI Dev Assistant 的最後一個產品功能（test generation），並透過這個過程展示完整的 Harness 演進

## 正文

### 1. 核心原則：踩坑驅動

大多數工程師在第一次嘗試建立 Agent 系統時，會本能地做一件事：先設計一套完美的 Harness，再開始使用 Agent。這個直覺是錯的。

原因在於：你不知道 Claude 會在你的具體情境下犯什麼錯，直到它真的犯了。每個專案的技術棧、工作流、團隊規範都不同，從別人的 Harness 抄來的規則，可能對應的是別人踩過、你永遠不會踩的坑。預防性設計帶來的是冗餘的約束，不是保護。

踩坑驅動的邏輯是：**先讓 Agent 裸跑，觀察犯什麼錯，再針對性加約束。**

「裸跑」不意味著魯莽。你仍然應該在低風險的任務上開始（讀取、分析、生成草稿），而非直接讓 Claude 在 production 環境裸跑。但在建立正式的 Harness 之前，你需要真實的踩坑數據，而不是想像中的踩坑清單。

**David Chu 的三個畢業信號**——以下三個信號出現時，代表是時候將踩坑提升為正式的 Harness 組件：

1. **CLAUDE.md 達到約 50 行**：代表有足夠的專案特定知識需要被記錄，值得投入時間整理成索引結構
2. **Claude 不再持續遵守某條規則**：prompt 建議開始失效，代表這條規則的違反後果足夠嚴重，需要升級為 Hook 層的強制約束
3. **到了定期審查的時間點**：每月一次的主動審計，不等踩坑才 review

每一條進入 Harness 的規則，都應該能回答這個問題：「這條規則對應哪一次真實發生的問題？」如果回答是「為了預防可能發生的問題」，這條規則在 Harness 中的地位是可疑的。

---

### 2. 六步流程

Harness 的建設是線性的，每一步都以前一步的產物為基礎。在 AI Dev Assistant 這個主案例中，以下六步就是你在模組 1-5 中實際走過的路——現在把它拆解清楚，目的是讓你能在下一個專案中重現這個過程。

**第 0 步：裸跑 + 記錄踩坑清單**

讓 Claude 在沒有任何 Harness 的情況下執行幾個真實任務。你的工作是觀察和記錄，而非干預。記錄格式建議：

```
日期 | 任務描述 | Claude 的錯誤行為 | 後果嚴重程度（低/中/高）
```

這份清單是後續所有 Harness 組件的原始資料。沒有這份清單，你加的每條規則都是猜測。

**第 1 步：建立最小 CLAUDE.md（50-100 行索引）**

裸跑幾次後，你會看到一些重複出現的問題：Claude 不知道這個專案用哪個測試框架、不清楚 API 路徑的命名規範、對 TypeScript 的嚴格模式設定不熟悉。這些是 CLAUDE.md 的素材。

50-100 行的 CLAUDE.md 應包含：
- 專案技術棧（Node.js 版本、TypeScript 設定、使用哪些主要套件）
- 目錄結構說明（`src/`、`tests/`、`docs/` 各放什麼）
- 最重要的幾條工作規範（例如：所有 API 回應必須包含 `requestId`）
- 深層文件的引用路徑（例如：「詳細 API 規範見 `docs/api-spec.md`」）

**第 2 步：加第一條約束（最頻繁或最嚴重的錯誤）**

從踩坑清單中挑出一條，標準是：後果最嚴重，或者出現頻率最高。不要同時加多條——你需要能夠評估每條約束的效果。

在 AI Dev Assistant 中，第一條 Hook 通常是攔截沒有跑測試就提交：一個 PreToolUse Hook，在 `git commit` 執行前確認 `npm test` 通過。

**第 3 步：加自驗證 Hook（Stop Validator）**

Stop Validator 幾乎對所有專案都有意義，因此它是標準的第 3 步，而非依賴踩坑。Stop Hook 的設計原則：

- 執行時間應在 60 秒以內（否則 Claude 的工作流會受到明顯干擾）
- 只驗證最關鍵的兩件事：測試通過、lint 無錯誤
- 失敗時輸出具體的錯誤訊息，讓 Claude 知道需要修什麼

```bash
#!/bin/bash
# .claude/hooks/stop-validator.sh
npm test --silent 2>&1
TEST_EXIT=$?

npm run lint --silent 2>&1
LINT_EXIT=$?

if [ $TEST_EXIT -ne 0 ] || [ $LINT_EXIT -ne 0 ]; then
  echo "Stop Validator: 測試或 lint 失敗，Claude 需修復後才能結束"
  exit 2
fi

echo "Stop Validator: 通過"
exit 0
```

**第 4 步：拆第一個 Subagent（context 過載時）**

當你開始注意到某類任務讓主對話的 context 快速膨脹——例如分析一個陌生的大型 codebase、系統性地審查安全漏洞——這是該委派給 Subagent 的信號。

第一個 Subagent 通常是「調查型」：給它一個問題和讀取權限，它在獨立 context 中工作，最終回傳一份精簡的分析報告。在 AI Dev Assistant 中，這就是 `agents/security-reviewer.md`。

**第 5 步：建立踩坑 → 規則閉環**

這是讓 Harness 持續進化的機制。每次踩坑後，完成一個完整的閉環：

1. 記錄踩坑（日誌）
2. 分析根因（Claude 為什麼會犯這個錯？）
3. 決定對應的 Harness 元件（Hook / Skill / CLAUDE.md 條目 / Agent 指令）
4. 實作並測試
5. 更新日誌，標記「已閉環」

閉環斷掉的最常見位置是第 3 步：踩坑被記錄了，但沒有人回頭分析並提煉為規則。這就是為什麼需要定期 review 機制（例如每週五固定 15 分鐘）。

**第 6 步：定期審計，移除過時組件**

每月執行一次。對每個 Harness 組件問三個問題：
- 這個組件最近 30 天有被觸發 / 使用嗎？
- 最新版本的 Claude 是否已內建處理了這個問題？
- 如果移除這個組件，最壞的情況是什麼？能接受嗎？

答案是「沒有」、「是」、「能接受」的組件，移除它。

---

### 3. 三種反模式

理解反模式的目的不是批評，而是讓你在看到自己的行為模式時能夠快速辨識並修正。

| 反模式 | 症狀 | 解法 |
|--------|------|------|
| 完美主義陷阱 | 花兩週設計 Harness 才開始用 Agent；Harness 文件寫得比實際使用時間更長 | 先跑再建。設定一個「不超過一個工作天」的規則：第一天裸跑，第二天才開始建第一個組件 |
| 約束堆積症 | 只加不刪，30 條規則互相衝突；Hook 目錄有 20 個腳本，沒有人清楚每個的作用 | 加一條新規則前，先問：有沒有一條舊規則可以因為這條新規則而被移除？每月審計是系統性解法 |
| 複製貼上症 | 把別人的 CLAUDE.md 或 Hook 直接複製進來；Harness 中有條目你自己說不清楚它對應的是什麼問題 | 每條規則必須能回答「這對應哪一次真實踩坑？」。無法回答的條目，刪除或標記為「待驗證」 |

---

## 實作範例：AI Dev Assistant Harness 完整演進（含 test generation 功能）

> 主案例延續：在 AI Dev Assistant 專案中，你已完成模組 1-5 的所有實作。這個範例做兩件事：回顧整個 Harness 演進的軌跡，並加入最後一個產品功能——test generation。

**目標**：
- 確認 AI Dev Assistant 的 Harness 已完整走完六步流程
- 加入 `src/test-gen.ts`（generateTests 函數），展示 Harness 在新功能開發中的實際作用
- 執行一次完整的第 6 步審計

**前置條件**：
- 已完成模組 1-5（CLAUDE.md、settings.json deny 規則、Stop Validator Hook、Security Reviewer Agent）
- Node.js 18.x 環境，`npm test` 可正常執行（Jest 29.x）
- `npm run lint` 可正常執行（ESLint 8.x + TypeScript 5.x）
- 專案根目錄存在 `src/` 和 `tests/` 目錄

**步驟**：

1. 回顧演進歷程——列出目前 `.claude/` 目錄的所有內容，確認六步流程的每個產物都存在：
   ```bash
   find .claude -type f | sort
   ```
   你應該看到類似這樣的輸出：
   ```
   .claude/agents/security-reviewer.md
   .claude/hooks/pre-commit-test.sh
   .claude/hooks/stop-validator.sh
   .claude/pitfalls.md
   .claude/settings.json
   CLAUDE.md
   ```
   如果某個產物缺失，在繼續之前先補齊。六步流程中任何一步的跳過，都代表有一個已知的風險點沒有被覆蓋。

2. 對照六步流程，確認每一步都有對應產物：
   - `CLAUDE.md`（50-100 行）→ 第 1 步完成
   - `.claude/settings.json`（包含 deny 規則）→ 第 2 步完成
   - `.claude/hooks/stop-validator.sh`（Stop Validator）→ 第 3 步完成
   - `.claude/agents/security-reviewer.md`（第一個 Subagent）→ 第 4 步完成
   - `.claude/pitfalls.md`（踩坑日誌，有已閉環的條目）→ 第 5 步完成
   有缺失的步驟，補齊後再繼續。

3. 現在加入最後一個產品功能。在 Claude Code 中使用以下 prompt：
   ```
   在 src/test-gen.ts 建立一個 generateTests 函數，符合以下規格：

   - 接收一個 TypeScript 檔案路徑（string）作為輸入
   - 使用 TypeScript compiler API 或 AST 解析，讀取該檔案所有具名 export 函數
   - 為每個函數生成一個 Jest 測試骨架，包含：
     - describe block（以模組名稱命名）
     - 每個函數對應一個 it block（描述為「should [函數名稱]」）
     - it block 內部包含一個 expect(true).toBe(true) 的佔位斷言，並附上 TODO 註解
   - 將生成的測試內容輸出為字串，不直接寫入檔案（由呼叫者決定輸出目標）
   - 在 tests/test-gen.test.ts 建立對應的測試，驗證至少兩個情境：
     1. 輸入包含單一 export 函數的 TypeScript 檔案，輸出應包含對應的 describe 和 it block
     2. 輸入不包含任何 export 函數的檔案，輸出應為空字串
   ```

4. 觀察 Stop Validator 是否如預期運作。當 Claude 完成實作並準備結束時，Stop Validator 應該自動執行 `npm test`。預期的驗證順序：
   - Claude 實作 `src/test-gen.ts` 和 `tests/test-gen.test.ts`
   - Claude 嘗試結束對話
   - Stop Validator 執行 `npm test && npm run lint`
   - 若測試通過，Stop Validator 回傳成功，Claude 正常結束
   - 若測試失敗，Stop Validator 回傳失敗，Claude 必須修復後重試
   如果 Stop Validator 沒有被觸發，檢查 `.claude/settings.json` 中的 hook 設定是否正確。

5. 執行第 5 步閉環。如果 Claude 在實作 `test-gen.ts` 的過程中犯了新的錯誤，將它記錄到踩坑日誌：
   ```bash
   # 開啟 .claude/pitfalls.md，在末尾新增一條
   # 格式：日期 | 任務 | 錯誤行為 | 後果嚴重程度 | 閉環方式（或待閉環）
   ```
   即使 Claude 表現完美，也應記錄「Claude 首次嘗試正確實作了 TypeScript AST 解析，不需要額外約束」——這是正面的數據，代表這類任務未來不需要 Hook 覆蓋。

6. 執行第 6 步審計。對 `.claude/hooks/` 目錄中的每個腳本，以及 `.claude/agents/` 中的每個 Agent，逐一回答以下問題：

   **對每個 Hook**：
   - 在過去 30 天內，這個 Hook 被觸發過嗎？（如果 Hook 腳本沒有寫 log，現在就加一行 `echo "[$(date)] Hook triggered: $HOOK_NAME" >> .claude/hook-activity.log`）
   - 這個 Hook 攔截的問題，Claude 4 或最新版本是否已經能自己避免？

   **對每個 Agent**：
   - 最近 30 天，這個 Agent 被委派過任務嗎？
   - Agent 的 prompt 是否仍然準確描述了它的工作範圍？（隨著主 codebase 演進，Agent 的指令可能已過時）

   **對 CLAUDE.md**：
   - 有沒有條目是三個月前加的，但從未被驗證有效？
   - 有沒有條目對應的問題，已被 Hook 或 ESLint 規則覆蓋，可以從 CLAUDE.md 移除（簡化索引）？

   記錄審計結果，移除確認無效的組件，更新確認仍然有效的組件的「最後驗證日期」。

**預期結果**：
- `src/test-gen.ts` 存在，包含 `generateTests` 函數，TypeScript 編譯無錯誤
- `tests/test-gen.test.ts` 存在，包含至少兩個測試情境，`npm test` 通過
- `.claude/pitfalls.md` 有新增條目（即使是「無新踩坑」的記錄）
- 審計後 `.claude/` 目錄結構精簡：沒有從未被觸發的 Hook，沒有描述已過時的 Agent
- 整個 `.claude/` 目錄的每個檔案，你都能說清楚它對應的是哪個踩坑或哪條原則

**常見錯誤**：

- **踩坑日誌沒有維護，閉環斷裂**：踩坑發生時有記錄，但沒有回頭提煉為規則，日誌變成無人閱讀的流水帳。解法是設定固定的 review 提醒——例如每週五下班前 15 分鐘開啟 `.claude/pitfalls.md`，處理「待閉環」的條目。這不是一個需要靈感的任務，而是一個需要紀律的習慣。

- **審計時捨不得刪，系統只增不減**：刪除 Hook 或 Agent 的心理阻力來自「萬一刪了之後又需要它呢？」。這個擔憂大多數時候是不成立的：Git 有版本控制，刪除的檔案可以找回來。更有用的問法是：「如果這個組件消失，最壞的情況是 Claude 犯一個我需要花多久時間修復的錯誤？」如果答案是「10 分鐘」，這個組件的存在成本（維護、認知負擔）很可能高於它提供的保護。

---

## 關鍵重點
- Harness 不是預防性設計，而是踩坑驅動的演進——每條規則都應該對應一個已發生的真實問題，而非假想的風險
- 六步流程的順序很重要：裸跑觀察在前，約束建立在後；Stop Validator 在基礎約束之後加入，Subagent 在 context 過載信號出現後才拆分
- 三種反模式中，「約束堆積症」最難察覺，因為它是緩慢積累的——每條規則加入時都有道理，問題在累積之後才會顯現
- 踩坑閉環是 Harness 保持有效的機制：日誌 → 分析 → 規則 → 驗證 → 標記閉環。任何一個環節斷掉，Harness 就會停止進化
- 定期審計（第 6 步）是 Harness 保持精簡的唯一機制——模型能力在進步，不做審計就等於讓 Harness 永遠對應舊版模型的問題

## 自我檢核
- 打開 `.claude/pitfalls.md`，裡面最老的「待閉環」條目是什麼時候加的？距今多少天？如果超過兩週，閉環機制已經斷了。
- 你的 Harness 中有沒有任何一條規則，你說不清楚它對應的是哪一次真實踩坑？這條規則是複製貼上症的症狀，需要重新評估它的必要性。
- 如果你明天將 Claude Code 升級到下一個主要版本，你打算如何驗證哪些 Hook 可以移除？這個驗證計畫如果你現在說不出來，第 6 步審計實際上還沒有真正落地。
