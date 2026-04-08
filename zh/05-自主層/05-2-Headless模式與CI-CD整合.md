# 05-2 Headless 模式與 CI/CD 整合
> 讓 Claude Code 在無人值守的環境中自動執行任務，從 PR review 到文件更新全部自動化。

## 學習目標
- 理解 Headless 模式（`-p` flag）的運作原理與三種輸出格式的差異
- 能夠用 `--session-id` 維持多輪無互動對話的上下文
- 設定 GitHub Actions 工作流程，讓每次 PR 自動觸發 Claude Code review
- 識別 CI/CD 整合中常見的權限與設定問題

## 正文

### 1. 為什麼需要 Headless 模式

到目前為止，Claude Code 的每一個操作都需要你坐在終端機前輸入指令、確認動作、閱讀輸出。這個互動模式在開發時非常直觀，但它有一個根本限制：你不在的時候，它什麼都不做。

然而有一整類任務，它們的最佳執行時機恰好是「你不在的時候」：

- 同事開了一個 PR，你在睡覺，但 PR 需要 code review
- 功能合併到 main，CI 應該自動更新 API 文件
- 每天凌晨三點掃描一次 security vulnerability，天亮前出報告

這些任務如果等人手動執行，要麼延遲要麼遺忘。Headless 模式（`-p` flag）解決這個問題：它讓 Claude Code 接受一個提示，執行完畢後退出，整個過程不需要任何人類互動。這讓 Claude Code 可以被任何自動化系統調用——shell 腳本、GitHub Actions、定時任務都可以。

### 2. `-p` 非互動模式

`-p` 是 `--print` 的縮寫，代表「執行並列印結果，然後退出」。

```bash
# 最基本的用法：問一個問題並取得答案
claude -p "解釋這個專案的架構"

# 以 JSON 格式輸出結構化結果，便於程式解析
claude -p "列出所有 API endpoints" --output-format json

# 串流 JSON：逐步輸出，適合需要即時顯示進度的場景
claude -p "分析 src/ 的程式碼品質" --output-format stream-json
```

三種輸出格式的選擇依據：

| 格式 | 用途 | 適合的場景 |
|------|------|------------|
| text（預設） | 純文字，人可直接閱讀 | 結果要顯示給使用者、寫入 log 檔 |
| json | 單一 JSON 物件，等任務完成後一次輸出 | 程式需要解析結果、寫入資料庫 |
| stream-json | 每個 chunk 是一個 JSON 物件，邊執行邊輸出 | 長時間任務需要顯示即時進度 |

在 shell 腳本中，`json` 格式讓你可以用 `jq` 精確提取欄位：

```bash
# 提取 Claude 分析結果中的 issues 陣列
claude -p "分析 src/auth.ts 的安全問題，以 JSON 回傳" \
  --output-format json | jq '.issues[]'
```

### 3. 多輪對話：用 `--session-id` 保持上下文

`-p` 模式預設是無狀態的：每次調用都是全新的 session，沒有記憶。但有些自動化工作流需要「先分析，再根據分析結果行動」，這需要跨調用保持上下文。

`--session-id` 參數讓多次 `-p` 調用共享同一個 session：

```bash
# 第一步：分析問題
claude -p "分析 src/auth.ts 中的 bug，列出所有問題" \
  --session-id fix-auth-session

# 第二步：在同一 session 中行動（Claude 記得第一步的分析結果）
claude -p "修復你剛才分析出的所有問題" \
  --session-id fix-auth-session

# 第三步：驗證結果
claude -p "確認修復後的 src/auth.ts 沒有遺留問題" \
  --session-id fix-auth-session
```

Session ID 是任意字串，你負責決定命名規則。建議用任務名稱加上日期或 PR 編號，例如 `fix-auth-2026-03-24` 或 `review-pr-142`，這樣 log 中的 session 記錄可讀。

### 4. GitHub Actions 整合

GitHub Actions 是 CI/CD 的標準工具，也是 Headless Claude Code 最常見的部署環境。Anthropic 提供了官方的 `claude-code-action`，直接封裝了鑑權、調用、輸出三個步驟。

**安裝方式一（推薦）**：在 Claude Code 互動 session 中執行斜線指令，它會引導你完成 GitHub App 安裝與權限設定：

```
/install-github-app
```

**安裝方式二（手動）**：直接在專案中建立 workflow 檔案，完全掌控設定細節（詳見下方實作範例）。

`claude-code-action` 支援兩種觸發模式：
- **自動觸發**：PR 開啟、push 到特定分支等 GitHub 事件自動執行
- **手動觸發（`@claude`）**：在 PR 或 issue 評論中 @claude，Claude 會回應並執行對應任務

### 5. 自動化場景與工作流設計

以下是四個實用的自動化場景，說明如何組合 GitHub 事件與 Claude Code 任務：

**PR 自動 Code Review**：
```
觸發：pull_request.opened / pull_request.synchronize
任務：分析 diff，針對功能正確性、安全問題、效能疑慮留下具體評論
輸出：直接在 PR 中留 review comments
```

**合併後自動更新文件**：
```
觸發：push to main
任務：比對 src/ 的變更，更新 README 中對應的 API 說明段落
輸出：commit 更新後的 README 到 main
```

**定時安全掃描**：
```
觸發：schedule（cron 表達式）
任務：掃描 package.json 的依賴，找出已知 CVE
輸出：在 GitHub Issues 開一張 security advisory ticket
```

**Lint 失敗自動修復**：
```
觸發：workflow_run（在 lint workflow 失敗後）
任務：讀取 lint 錯誤，修復所有可自動修復的規則違反
輸出：commit 修復後的檔案到同一分支
```

設計自動化工作流時，有兩個原則值得記住：第一，讓 Claude 的任務描述盡量具體，「review this PR」比「analyze the changes and provide specific line-level feedback on functionality, security, and performance」的結果品質差很多。第二，Claude 在 CI 環境中操作的是真實的 repo，要設定最小必要權限，不要給 `contents: write` 除非任務確實需要寫入。

## 實作範例：GitHub Action 自動 Code Review
> 主案例延續：在 AI Dev Assistant 專案中，設定每次 PR 開啟時自動執行 code review，讓 Claude 在 PR 中留下具體的評論。

**目標**：設定 GitHub Actions workflow，每次有人開啟或更新 ai-dev-assistant 的 PR 時，Claude 自動分析 diff 並在 PR 中留下結構化的 review 評論。

**前置條件**：
- 已完成 03-3（/review Skill 設定）
- ai-dev-assistant 已推送到 GitHub，並有對應的 remote repo
- 你有該 repo 的 admin 或 secrets 管理權限
- 持有有效的 Anthropic API key

**步驟**：

1. 在 Claude Code 互動 session 中嘗試官方安裝指令（若成功可跳過步驟 2-3）：
   ```
   /install-github-app
   ```
   按照終端的提示完成 GitHub App 授權流程。若遇到網路限制或需要更細緻的控制，繼續步驟 2。

2. 手動建立 workflow 目錄與檔案：
   ```bash
   mkdir -p /c/projects/ai-dev-assistant/.github/workflows
   ```

3. 建立 `.github/workflows/claude-review.yml`，內容如下：
   ```yaml
   name: Claude Code Review
   on:
     pull_request:
       types: [opened, synchronize]

   jobs:
     review:
       runs-on: ubuntu-latest
       permissions:
         contents: read
         pull-requests: write
       steps:
         - uses: actions/checkout@v4
           with:
             fetch-depth: 0
         - uses: anthropics/claude-code-action@v1
           with:
             anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
             prompt: |
               Review this PR for the following aspects:
               1. Functionality correctness - does the logic achieve what the PR describes?
               2. Security issues - any hardcoded secrets, injection risks, or unsafe operations?
               3. Performance concerns - unnecessary loops, missing indexes, or expensive operations?
               4. Code quality - TypeScript type safety, naming clarity, ESLint rule compliance
               Provide specific line-level comments for each issue found.
               For each comment, include the severity (critical / warning / suggestion).
   ```

4. 在 GitHub repo 中設定 API key secret：進入 repo 頁面 → Settings → Secrets and variables → Actions → New repository secret，名稱填 `ANTHROPIC_API_KEY`，值填你的 API key。

5. 將 workflow 檔案推送到 repo：
   ```bash
   cd /c/projects/ai-dev-assistant
   git add .github/workflows/claude-review.yml
   git commit -m "ci: add Claude Code automatic PR review workflow"
   git push origin main
   ```

6. 建立一個測試 PR。從 main 開一個分支，隨意修改一個 TypeScript 檔案，推送並開 PR：
   ```bash
   git checkout -b test/trigger-claude-review
   echo "// test comment" >> src/index.ts
   git add src/index.ts
   git commit -m "test: trigger Claude review workflow"
   git push origin test/trigger-claude-review
   ```
   然後在 GitHub 上對這個分支開一個 PR。

7. 在 GitHub repo 的 Actions 分頁觀察 workflow 執行狀態。成功時，約 30-60 秒後在 PR 的 Files Changed 分頁會出現 Claude 的 review 評論。

8. 驗證評論格式符合預期：每個評論應包含問題類型、具體描述、以及 severity 標記（critical / warning / suggestion）。

**預期結果**：
- PR 開啟後，GitHub Actions 自動觸發 `Claude Code Review` workflow
- workflow 在 Actions 頁面顯示綠色（成功）狀態
- PR 的 Files Changed 頁面出現 Claude 留下的 review 評論，每條評論有明確的 severity 和具體的修改建議
- 若 PR 沒有問題，Claude 也會留下「No critical issues found」的確認評論

**常見錯誤**：
- **`Error: ANTHROPIC_API_KEY is not set` 導致 workflow 失敗**：在 Actions 頁面的 workflow log 中可以看到這個錯誤。解法：確認你在 Settings → Secrets and variables → Actions 下新增了名稱完全正確的 secret（大小寫敏感）。設定 secret 後，需要重新觸發 workflow（例如 push 一個新 commit 到 PR 分支）。
- **Workflow 成功但 PR 中沒有出現評論**：最常見的原因是 workflow 缺少 `pull-requests: write` 權限。檢查 `permissions` 區塊，確認 `pull-requests: write` 存在。另一個可能是 `actions/checkout` 沒有加 `fetch-depth: 0`，導致 Claude 無法取得完整的 diff。

## 關鍵重點
- `-p` flag 讓 Claude Code 單次執行後退出，是所有自動化整合的基礎
- `--session-id` 讓多次 `-p` 調用共享 context，實現「分析後行動」的兩步驟自動化
- `anthropics/claude-code-action@v1` 封裝了 CI 環境中的鑑權與調用，是 GitHub Actions 整合的最短路徑
- GitHub Actions 中 Claude 需要的最低權限是 `contents: read` 加 `pull-requests: write`，不要超額授權
- Prompt 越具體，CI 中的 review 品質越穩定；「review this PR」是最差的 prompt，列出具體檢查項目才能得到可操作的結果

## 自我檢核
- 你能說出 `--output-format json` 和 `--output-format stream-json` 的使用時機差異，並舉出一個適合各自的具體場景嗎？
- 如果 GitHub Actions 中的 Claude review workflow 執行成功但 PR 沒有任何評論，你會依序檢查哪三個地方？
- 用 `--session-id`，你能設計一個三步驟的自動化腳本（分析 → 修復 → 驗證）嗎？

---

⬅️ [上一章：Agent Teams](05-1-Agent-Teams.md) ｜ 📖 [回目錄](../00-索引與摘要.md) ｜ ➡️ [下一章：Agent SDK](05-3-Agent-SDK.md)

🌐 [English Version](../../en/05-autonomous/05-2-headless-cicd.md)
