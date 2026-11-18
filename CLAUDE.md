# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

紫微斗數十四主星拓撲盤 — an interactive SVG visualization of Zi Wei Dou Shu (紫微斗數) star topology on the cyclic group Z₁₂. The entire application is a **single self-contained `index.html`** (inline CSS + vanilla JS, no dependencies, no build step) designed for direct GitHub Pages deployment.

UI text and domain terminology are in Traditional Chinese (zh-TW). Respond to the user in zh-TW.

## Development

No build, lint, or test tooling exists. To develop:

```bash
open index.html                # or:
python3 -m http.server 8000    # then visit http://localhost:8000
```

The only external resources are Google Fonts (Noto Serif TC / Noto Sans TC); everything else works offline.

## Architecture (all inside index.html's single IIFE)

The script is organized in layers, top to bottom:

1. **Domain data tables** — `ZW_OFF` / `TF_OFF` (fixed offsets of the 紫微 and 天府 star series), `SIHUA` (四化 by year stem, includes 昌曲輔弼; follows the user's reference book, where 庚 is 陽武同陰 — see 研究紀錄.md §0.1 before changing), `LUCUN`, `KUIYUE`, `HUOLING_BASE` (吉煞 star lookup tables keyed by birth pillars).
2. **Position math** — everything is arithmetic mod 12 (`mod12`, `theta`, `pt`). Core invariant in `starPos(t)`: the 紫微 series rotates as `t + offset` while the 天府 series counter-rotates as `4 - t + offset` (mirror across the 寅申 axis, i.e. 紫微+天府 ≡ 4). `auxPos()` computes 吉煞 stars from year stem/branch, month, and hour (天馬 uses the birth month's branch, 正月 = 寅) — these are anchored to birth pillars and deliberately do NOT rotate with the chart number (局數).
3. **SVG rendering layers** — a static base plate, then several `<g>` groups redrawn independently: `gAxis` (mirror axes), `gGeo` (branch-level overlays that do not depend on the chart number: the 六合梯 path and the two 三合稜柱, drawn by `drawGeo`), `gEdges` (三合/對宮/暗合 relationship lines via `edgesAt`/`edgeStyle`/`edgeVisible`), `gStars` (14 main stars + 四化 badges), `gAux` (吉煞 symbols). Checkbox toggles map 1:1 to edge/layer visibility.
4. **Animation/controls** — `goTo()` tweens rotation between the 12 charts with `requestAnimationFrame` (respects `prefers-reduced-motion`); `schedule()` drives auto-play.

Relationship semantics used throughout: 三合 = position difference 4 or 8; 對宮 = difference 6; 暗合 = position sum ≡ 1 (mod 12).

The 「數學觀察」 card in the HTML documents the mathematical theorems the visualization demonstrates (鏡射定理, 互斥定理, 2:4:2:4律, etc.) — keep it consistent with the code if the math logic changes.

## Git Conventions

Commit messages follow the parent vault convention (English, conventional commits style), though existing history contains Chinese messages.

## Research notes

`研究紀錄.md` records research scope decisions, negative results, and the main line of inquiry (why the two star series are arranged as they are, and how that ties to the palace naming). Append new findings there; only brute-force-verified statements go into the 「數學觀察」 card.
