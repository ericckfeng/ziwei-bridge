# Resume Context — 平滑參數動畫與宮位對比強化

## 狀況摘要

上一個時段因 rate limit 中斷，尚未執行任何 Task。下一個時段直接接 Task 1 開始。

## 關鍵路徑

```
spec   → docs/superpowers/specs/2026-07-05-smooth-param-animation-design.md
plan   → docs/superpowers/plans/2026-07-05-smooth-param-animation.md
ledger → .superpowers/sdd/progress.md
目標檔 → index.html（單一檔，在 repo 根目錄）
分支   → main（直接在 main 上 commit）
```

## 5 個 Task 摘要

| Task | 名稱 | 核心改動 |
|------|------|----------|
| 1 | 宮位色彩對比 | --yin 色、扇形 opacity .08→.16、宮界線 .15→.4、文字 opacity 1 |
| 2 | 吉煞層狀態驅動重構 | 新增 ptDeg/auxRenderState/currentParams，drawAux(state) 取代清空重畫，onParamChange() 立即版 |
| 3 | 圓弧補間動畫 | onParamChange() 動畫版：最短圓弧 rAF 補間 700ms，重入取消 |
| 4 | 四化徽章飛行 | gFly 圖層、huaFlight 旗標、starCoordFor/renderFly，年干改變時 4 顆徽章飛行 |
| 5 | 循環選單 | cycleSel 選單、CYCLE 表、stepCycle(dir)，◀▶/自動運行泛化到 5 個輸入參數 |

## 執行方式（subagent-driven-development）

每個 Task 的流程：
1. 用 `scripts/task-brief PLAN N` 萃取 brief（Task 1 的 brief 已存在：.superpowers/sdd/task-1-brief.md）
2. 記錄當前 HEAD 作為 BASE commit（e141f21 是 Task 1 的 BASE）
3. 派發 implementer subagent（model: haiku 適合 Task 1/2/3/5；Task 4 用 sonnet）
4. 用 `scripts/review-package BASE HEAD` 產生 diff 檔
5. 派發 task reviewer subagent 審查
6. 如有 Critical/Important 問題，派發 fix subagent
7. 更新 ledger

## implementer subagent prompt 範本路徑

/Users/jkfeng/.claude/plugins/cache/claude-plugins-official/superpowers/6.1.1/skills/subagent-driven-development/implementer-prompt.md

## scripts 路徑

/Users/jkfeng/.claude/plugins/cache/claude-plugins-official/superpowers/6.1.1/skills/subagent-driven-development/scripts/task-brief
/Users/jkfeng/.claude/plugins/cache/claude-plugins-official/superpowers/6.1.1/skills/subagent-driven-development/scripts/review-package

## Global Constraints（每個 Task 都適用）

- 全部程式碼在 index.html 單檔（inline CSS + IIFE script），不得引入外部函式庫。
- UI 文字繁體中文（zh-TW）。
- 無測試框架；驗證為瀏覽器手動實測（python3 -m http.server 8000）。
- 動畫 700ms、easing 1-(1-k)^3、prefers-reduced-motion 時直接落位。
- 局數旋轉時吉煞星保持固定不動。
- Commit message 英文 conventional commits。

## Task 1 implementer dispatch 範本

以下可直接複製貼上（注意 model 用 haiku，brief 已萃取）：

```
You are implementing Task 1: 宮位色彩對比強化 (palace color contrast).

Read your task brief first:
/Users/jkfeng/Documents/Academic_Life_Vault/01_Projects/Dev/ziwei-bridge/.superpowers/sdd/task-1-brief.md

Context: The project is a single self-contained index.html (Zi Wei Dou Shu chart, vanilla JS + SVG, zero dependencies) at /Users/jkfeng/Documents/Academic_Life_Vault/01_Projects/Dev/ziwei-bridge/. This is a pure CSS/attribute tweak with exact before→after code given in the brief. Locate lines by content if line numbers drifted.

Global Constraints:
- All code inside index.html, no external libraries.
- Commit message exactly as given in brief Step 5.
- Verify by re-reading edited regions (git diff should show only these changes).

Work from: /Users/jkfeng/Documents/Academic_Life_Vault/01_Projects/Dev/ziwei-bridge

Report to: /Users/jkfeng/Documents/Academic_Life_Vault/01_Projects/Dev/ziwei-bridge/.superpowers/sdd/task-1-report.md

Report back with:
- Status: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT
- Commits (short SHA + subject)
- One-line verification summary
- Concerns if any
- Report file path
```

## 下一個時段恢復步驟

1. 開啟新的 Claude Code session，cd 到 repo 根目錄。
2. 閱讀 .superpowers/sdd/progress.md 確認哪個 task 要繼續。
3. 從 Task 1 開始，用上面的 dispatch 範本派發 implementer（Task 1 brief 已存在，BASE = e141f21）。
4. 之後的 task 用 `scripts/task-brief PLAN N` 萃取，並記錄各自的 BASE commit。
5. 每個 task 結束後更新 ledger。
