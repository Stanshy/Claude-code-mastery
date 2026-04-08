# 04-3 平行開發與 Worktree

> 用 Git Worktree 同時跑多個 Claude session，將等待時間轉換為開發產能。

## 學習目標

- 理解 Claude Code 的 Agent Loop 為何需要平行化策略
- 掌握 Git Worktree 的原理與 `claude --worktree` 指令的運作方式
- 能夠建立多個獨立 session 同時開發不同功能
- 了解 Writer/Reviewer 模式與 Subagent 隔離的應用場景

## 正文

### 1. 為什麼需要平行開發

Claude Code 的 Agent Loop 在執行一項任務時，會持續讀取檔案、寫入程式碼、執行測試，直到任務完成為止。這個過程是序列的：你下指令，等待結果，看結果，再下指令。一個中等複雜度的功能，往返時間可能是 3 到 5 分鐘，有時更長。

如果你只開一個 terminal session，這 3 到 5 分鐘就是純粹的等待。乘上一天 20 個任務，你有將近 1 到 2 小時的時間是空閒的。

工程師 Boris Cherny 採用的策略是同時跑 5 個本地 session 加上 5 到 10 個線上 session，以系統通知協調何時切換注意力。30 天之內，他完成了 259 個 Pull Request。這個數字的關鍵不是速度，而是並行度：每一個 session 都在等待的時候，另外 9 個 session 正在工作。

平行開發的前提是：各個 session 的工作空間必須互相隔離。這就是 Git Worktree 存在的原因。

### 2. Git Worktree 原理

Worktree 是 Git 的原生功能，不需要額外安裝。它允許你在同一個 repository 下建立多個工作目錄，每個目錄有獨立的 branch 和檔案狀態，但共用同一份 `.git` 歷史與物件資料庫。

用一句話說：你可以在同一個 repo 同時 checkout 多個不同的 branch，每個 branch 在自己的目錄裡獨立運作。

`claude --worktree feature-name` 這個指令做了以下事情：

1. 在 repo 根目錄之外建立一個新目錄，名稱對應你指定的功能名稱
2. 建立一個同名的新 branch，並 checkout 到該目錄
3. 在這個隔離的工作目錄中啟動一個完整的 Claude Code session

這個設計意味著每個 Claude session 可以自由修改檔案、建立新檔案、執行 git commit，完全不影響主分支或其他 worktree 中的工作。

### 3. 平行開發工作流

典型的平行開發配置如下：

```
終端 1：claude --worktree feature-auth      → 開發登入功能
終端 2：claude --worktree feature-dashboard → 開發儀表板
終端 3：claude -c                           → 主分支日常工作
```

三個 session 同時運行，彼此完全隔離。即使 session 1 和 session 2 都修改了 `src/utils.ts`，它們各自在不同的 branch 上工作，不會互相覆寫。

切換注意力的節奏如下：在 session 1 下完指令後，立刻切換到 session 2 查看上一個任務的結果或下新指令。當兩個 session 都在跑時，你可以在 session 3 處理 code review 或文件工作。每個 session 完成後，系統通知會告知你可以回來看結果。

這個工作流的本質是將「等待 Claude 回應」的時間，替換成「處理其他 session 的工作」的時間。

### 4. Writer/Reviewer 模式

Worktree 也支援一種特殊的協作模式，讓兩個 Claude session 扮演不同角色：

| Session A（Writer） | Session B（Reviewer） |
|--------------------|-----------------------|
| 在 worktree 中實作功能 | 在主分支或另一個 worktree 中審查實作 |
| 根據 Reviewer 的回饋修改程式碼 | 確認修正是否完整，找出 edge cases |
| 完成後發出 PR 或合併請求 | 給出最終核准意見 |

Writer session 專注在實作，不需要分心做品質把關。Reviewer session 不需要了解實作細節，只需要閱讀現有程式碼並提出問題。這種分工讓兩個 session 的輸出都更專注、品質更高。

實際操作上，你可以在 Reviewer session 的 prompt 中明確說明：「這是 feature-auth worktree 中的 auth.ts，請找出所有未處理的錯誤情境和缺少的邊界條件」，讓 Claude 扮演純粹的審查角色。

### 5. Subagent 的 Worktree 隔離

當你在 Claude Code 中使用 Agent 系統（orchestrator 呼叫 subagent）時，subagent 預設在同一個工作目錄中執行。如果多個 subagent 同時操作同一批檔案，結果會互相覆寫。

解決方式是在 Agent frontmatter 中加入 `isolation: worktree` 設定。加上這個設定後，orchestrator 在呼叫每個 subagent 時，會自動為它建立一個獨立的 worktree，subagent 在其中工作完成後，orchestrator 再收集結果合併。

這個機制讓你可以安全地讓多個 subagent 平行修改程式碼，而不需要人工協調寫入順序。

## 實作範例：AI Dev Assistant 平行開發 auth 和 dashboard

> 主案例延續：在 AI Dev Assistant 專案中，我們用兩個 worktree session 同時開發登入驗證和儀表板兩個功能模組。

**目標**：建立兩個獨立的 worktree session，平行開發 `auth.ts` 和 `dashboard.ts`，最終合併回 main 分支。

**前置條件**：已完成 04-2 的內容；`ai-dev-assistant` 專案已初始化 git，且至少有一個 commit（`git log` 可以看到 history）。

**步驟**：

1. 在 `ai-dev-assistant` 目錄中確認 git 狀態：
   ```bash
   git status
   git log --oneline -3
   ```
   確認目前在 `main` 分支，且工作目錄是乾淨的（no changes to commit）。

2. 開啟第一個終端，啟動 auth worktree session：
   ```bash
   claude --worktree feature-auth
   ```
   Claude 會自動建立 `feature-auth` branch 並在新目錄中啟動 session。

3. 在 session 1 中下達指令：
   ```
   在 src/ 目錄下建立 auth.ts，實作一個 validateToken 函數。
   函數接受一個字串 token，回傳 { valid: boolean; userId?: string }。
   如果 token 為空或長度小於 10，回傳 { valid: false }。
   ```

4. 不要等待 session 1 完成，立刻開啟第二個終端，啟動 dashboard worktree session：
   ```bash
   claude --worktree feature-dashboard
   ```

5. 在 session 2 中下達指令：
   ```
   在 src/ 目錄下建立 dashboard.ts，實作一個 getMetrics 函數。
   函數回傳 { activeUsers: number; requestsPerMinute: number; errorRate: number }。
   初始值全部為 0，並加上一行 JSDoc 說明每個欄位的含義。
   ```

6. 兩個 session 現在平行工作。你可以在兩個終端之間切換，觀察各自的進度，或在一個 session 完成時立刻給下一個指令，同時讓另一個繼續跑。

7. 兩個 session 都完成後，回到主分支進行合併：
   ```bash
   git checkout main
   git merge feature-auth
   git merge feature-dashboard
   ```

8. 確認兩個檔案都已存在於 main 分支：
   ```bash
   ls src/
   # 預期看到 auth.ts 和 dashboard.ts
   ```

9. 清理已完成的 worktree（Claude 在 session 結束時會自動清理，也可以手動執行）：
   ```bash
   git worktree list
   git worktree remove feature-auth
   git worktree remove feature-dashboard
   ```

**預期結果**：
- `feature-auth` 和 `feature-dashboard` 兩個 branch 各自有獨立的 commit history
- 兩個 session 在開發過程中的檔案修改完全隔離，互不影響
- 合併到 `main` 後，`src/auth.ts` 和 `src/dashboard.ts` 都存在，功能完整

**常見錯誤**：

- **`fatal: 'feature-auth' is already checked out at '/path/to/worktree'`**：同一個 branch 名稱只能在一個 worktree 中被 checkout。如果你先前建立過同名的 worktree 但沒有清理，就會出現這個錯誤。解決方式：執行 `git worktree list` 查看現有的 worktree，用 `git worktree remove` 移除舊的，或改用不同的 branch 名稱。

- **合併時出現衝突（`CONFLICT (content): Merge conflict in src/utils.ts`）**：如果兩個 session 都修改了同一個共用檔案（例如 `utils.ts` 或 `index.ts`），合併時就會發生衝突。這和一般的 git merge 衝突完全相同，用 `git status` 確認衝突檔案，手動編輯解決後執行 `git add` 和 `git commit`。預防方式是在開始前明確劃分各 session 負責的檔案範圍，避免兩個 session 同時碰觸同一個檔案。

## 關鍵重點

- Git Worktree 是 Git 原生功能，讓同一個 repo 同時 checkout 多個 branch 到不同目錄，每個目錄有完全獨立的檔案狀態
- `claude --worktree feature-name` 自動建立 worktree 並在其中啟動隔離的 Claude session，無需手動操作 git
- 平行開發的核心價值是將 Agent Loop 的等待時間轉換為其他 session 的工作時間，而不是純粹靠速度取勝
- Writer/Reviewer 模式讓兩個 session 分工明確，各自專注在實作或審查，避免同一個 session 既寫又審造成的盲點
- 同一個 branch 名稱不能同時出現在兩個 worktree 中，每個 session 必須使用唯一的 branch 名稱

## 自我檢核

- 你有辦法在不關閉第一個 Claude session 的情況下，同時啟動第二個 session 並開始工作嗎？實際試試看，確認兩個 worktree 的 `git branch` 顯示的是不同的 branch 名稱。
- 如果兩個 session 都修改了 `src/config.ts`，合併時會發生什麼事？你能用 `git diff feature-auth feature-dashboard -- src/config.ts` 在合併前就預覽差異嗎？
- Boris Cherny 用 5 個本地 session 完成 259 個 PR 的策略，其瓶頸在哪裡？是 Claude 的速度、你切換注意力的速度，還是 git merge 的頻率？

---

⬅️ [上一章：上下文管理策略](04-2-上下文管理策略.md) ｜ 📖 [回目錄](../00-索引與摘要.md) ｜ ➡️ [下一章：Agent Teams](../05-自主層/05-1-Agent-Teams.md)

🌐 [English Version](../../en/04-orchestration/04-3-worktree.md)
