# Claude Code Mastery：從入門到精通

Claude Code 完整教學講義——涵蓋基礎操作、配置系統、自動化、Agent 編排、Harness Engineering，以及頂級工程師的實戰框架。

> **[English Version (README.en.md)](README.en.md)**

---

## 課程結構：五層進化模型

本課程以 David Chu 的五層進化模型為骨架，每個模組對應一個進化層級：

```
第一層 對話式  →  模組 1：認識 Claude Code
第二層 配置式  →  模組 2：基礎配置
第三層 自動化  →  模組 3：Hooks / Skills / MCP
第四層 編排式  →  模組 4：Subagents / 上下文管理 / 平行開發
第五層 自主式  →  模組 5：Agent Teams / Headless / SDK
方法論層      →  模組 6：Harness Engineering
案例層        →  模組 7：Boris Cherny / gstack / 開源案例
參考層        →  模組 8：速查表 + 資源索引
```

| 模組 | 主題 | 章數 |
|------|------|------|
| 1 | [認識 Claude Code](01-認識Claude-Code/) | 2 |
| 2 | [基礎配置](02-基礎配置/) | 3 |
| 3 | [自動化層](03-自動化層/) | 4 |
| 4 | [編排層](04-編排層/) | 3 |
| 5 | [自主層](05-自主層/) | 3 |
| 6 | [Harness Engineering](06-Harness-Engineering/) | 2 |
| 7 | [實戰框架與工作流](07-實戰框架與工作流/) | 4 |
| 8 | [參考資源](08-參考資源/) | 2 |

共 **8 個模組、23 章**。完整索引見 [00-索引與摘要](00-索引與摘要.md)。

---

## 主案例專案：AI Dev Assistant

全書透過一個貫穿的專案展示所有概念。雙線演進，避免產品複雜度拖垮學習：

**產品功能線**（專案加新功能）：01-2 建立骨架 → 03-3 code review → 05-3 bug fix → 06-2 test generation

**架構能力線**（學 Claude Code 能力）：CLAUDE.md → 設定與權限 → Hooks → Skills → MCP → Subagents → 上下文管理 → Worktree → Agent Teams → CI/CD

技術棧：Node.js 18.x / TypeScript 5.x / Jest 29.x / ESLint 8.x — 詳見 [PROJECT_SPEC.md](PROJECT_SPEC.md)

內容生成規範：[CONTENT_RULES.md](CONTENT_RULES.md) / [SYSTEM_PROMPT.md](SYSTEM_PROMPT.md)

---

## 章節總覽

### 模組 1：認識 Claude Code

- [01-1 什麼是 Claude Code](01-認識Claude-Code/01-1-什麼是Claude-Code.md) — Agentic CLI 的定義、Agent Loop、工具對比
- [01-2 安裝與第一次成功](01-認識Claude-Code/01-2-安裝與第一次成功.md) — 安裝、認證、建立專案骨架

### 模組 2：基礎配置

- [02-1 CLAUDE.md 完全指南](02-基礎配置/02-1-CLAUDE-MD完全指南.md) — Boris Cherny 哲學、撰寫規範、大小控制
- [02-2 設定檔與權限系統](02-基礎配置/02-2-設定檔與權限系統.md) — settings.json、五種權限模式、allow/deny 語法
- [02-3 Agent = Model + Harness](02-基礎配置/02-3-Agent等於Model加Harness.md) — Harness 核心概念預告

### 模組 3：自動化層

- [03-1 Hooks 基礎](03-自動化層/03-1-Hooks基礎.md) — 確定性 vs 建議性、6 個核心事件、配置格式
- [03-2 Hooks 進階與 Validator 模式](03-自動化層/03-2-Hooks進階與Validator模式.md) — Stop Hook Validator Pattern、四種處理器類型
- [03-3 Skills 系統](03-自動化層/03-3-Skills系統.md) — 按需載入、SKILL.md 格式、動態內容注入
- [03-4 MCP Servers](03-自動化層/03-4-MCP-Servers.md) — 外部工具連接、GitHub MCP、Tool Search

### 模組 4：編排層

- [04-1 自訂 Subagents](04-編排層/04-1-自訂Subagents.md) — 獨立上下文、角色定義、三種調用方式
- [04-2 上下文管理策略](04-編排層/04-2-上下文管理策略.md) — 8 個管理技巧、Document & Clear 法
- [04-3 平行開發與 Worktree](04-編排層/04-3-平行開發與Worktree.md) — Git Worktree、Boris 的 5+5 模式

### 模組 5：自主層

- [05-1 Agent Teams](05-自主層/05-1-Agent-Teams.md) — Team Lead + Teammates 協作架構
- [05-2 Headless 模式與 CI/CD 整合](05-自主層/05-2-Headless模式與CI-CD整合.md) — 非互動模式、GitHub Actions
- [05-3 Agent SDK](05-自主層/05-3-Agent-SDK.md) — Python/TypeScript SDK、程式化控制

### 模組 6：Harness Engineering

- [06-1 七大設計原則](06-Harness-Engineering/06-1-七大設計原則.md) — 上下文架構、架構約束、自驗證、隔離、熵管理、可拆卸性、Agent 可讀性
- [06-2 從零建立 Harness](06-Harness-Engineering/06-2-從零建立Harness.md) — 踩坑驅動六步流程、三種反模式

### 模組 7：實戰框架與工作流

- [07-1 Boris Cherny 工作法](07-實戰框架與工作流/07-1-Boris-Cherny工作法.md) — 30 天 259 PR 的核心方法
- [07-2 gstack 角色系統](07-實戰框架與工作流/07-2-gstack角色系統.md) — YC 總裁的 18 Skill 工程團隊
- [07-3 開源 Harness 案例](07-實戰框架與工作流/07-3-開源Harness案例.md) — 三個頂級專案的設計分析
- [07-4 David Chu 進化框架](07-實戰框架與工作流/07-4-David-Chu進化框架.md) — 五層模型、三條畢業路徑

### 模組 8：參考資源

- [08-1 速查表](08-參考資源/08-1-速查表.md) — 指令、快捷鍵、Hook 事件、權限語法
- [08-2 資源索引](08-參考資源/08-2-資源索引.md) — 精選 37 個資源 + Top 10 必收藏

---

## 建議閱讀順序

| 讀者背景 | 建議路線 |
|----------|----------|
| 從未用過 Claude Code | 模組 1 → 2 → 3 → 4 → 5 |
| 已會基本操作，想進階 | 模組 3 → 4 → 6 → 7 |
| 想學 Harness Engineering | 模組 6 → 7 → 3（回頭對照） |
| 時間有限，只看 3 章 | 01-1 → 02-1 → 03-2 |

---

Stanshy | 2026-03
