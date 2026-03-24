# AI Dev Assistant — 專案規格

## 定位

一個用 Claude Code 建構的 CLI 工具，能自動執行 code review、修 bug、分析 PR、生成測試。
貫穿全書，每章為它增加一層能力。

---

## 技術棧

| 項目 | 技術 | 版本 |
|------|------|------|
| 語言 | Node.js（TypeScript） | Node.js 18.x LTS / TypeScript 5.x |
| 測試 | Jest | 29.x |
| Lint | ESLint + Prettier | ESLint 8.x / Prettier 3.x |
| 版本控制 | Git | — |
| 套件管理 | npm | — |
| CI/CD | GitHub Actions | — |

---

## 核心功能

1. **自動 code review**：分析 git diff，產出結構化報告（功能 / 安全 / 效能 / 可維護性）
2. **自動修 bug**：定位問題 → 修復 → 跑測試驗證
3. **PR 分析**：摘要、風險評估、改進建議
4. **測試生成**：讀取 source 檔案 → 生成對應的 Jest 測試

---

## 專案結構

```
ai-dev-assistant/
├── src/
│   ├── index.ts          # CLI 入口
│   ├── reviewer.ts       # code review 邏輯
│   ├── fixer.ts          # bug 修復邏輯
│   ├── analyzer.ts       # PR 分析邏輯
│   └── test-gen.ts       # 測試生成邏輯
├── tests/
│   ├── reviewer.test.ts
│   ├── fixer.test.ts
│   └── analyzer.test.ts
├── scripts/
│   └── validate.sh       # Hook 用的驗證腳本
├── .claude/
│   ├── settings.json
│   ├── settings.local.json
│   ├── agents/
│   │   ├── security-reviewer.md
│   │   └── debugger.md
│   ├── skills/
│   │   └── review/
│   │       └── SKILL.md
│   └── hooks/
│       ├── protect-files.sh
│       └── stop-validator.sh
├── .github/
│   └── workflows/
│       └── claude-review.yml
├── CLAUDE.md
├── .mcp.json
├── package.json
├── tsconfig.json
└── .gitignore
```

---

## CLI 介面

```bash
assistant "修復 login 模組的 bug"
assistant "review 這個 PR"
assistant "分析 #42 PR 的風險"
assistant "為 src/reviewer.ts 生成測試"
```

---

## 雙線演進

### 產品功能線（專案什麼時候加新功能）

| 章節 | 產品進度 |
|------|---------|
| 01-2 | 建立 CLI 骨架（package.json + src/ + tests/） |
| 03-3 | 加 code review 功能 |
| 05-3 | 加 bug fix 功能 |
| 06-2 | 加 test generation 功能 |

### 架構能力線（每章學什麼 Claude Code 能力）

| 章節 | 能力 |
|------|------|
| 02-1 | CLAUDE.md |
| 02-2 | 設定檔與權限 |
| 03-1 | Hooks 基礎 |
| 03-2 | Hooks 進階（Validator） |
| 03-4 | MCP Servers |
| 04-1 | Subagents |
| 04-2 | 上下文管理 |
| 04-3 | Worktree 平行開發 |
| 05-1 | Agent Teams |
| 05-2 | Headless + CI/CD |
