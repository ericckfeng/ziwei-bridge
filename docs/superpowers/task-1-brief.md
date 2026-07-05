### Task 1: 宮位色彩對比強化

**Files:**
- Modify: `index.html`（`:root` CSS 變數 ~line 16；`base()` IIFE ~lines 242-269）

**Interfaces:**
- Produces: 無 JS 介面變動，純視覺。

- [ ] **Step 1: 調整 `--yin` 色**

`:root` 中（~line 16）：

```css
  --yang:#f2913d; --yin:#4f7cd9;
```

改為：

```css
  --yang:#f2913d; --yin:#3f6ad4;
```

- [ ] **Step 2: 提高扇形底色不透明度與宮界線**

`base()` 內（~lines 250-252）：

```js
    el("path",{d:`M${p1} L${p2} A${R_WOUT},${R_WOUT} 0 0 1 ${p3} L${p4} A${R_WIN},${R_WIN} 0 0 0 ${p1} Z`,
      fill:i%2===0?"var(--yang)":"var(--yin)",opacity:.08,
      stroke:"rgba(217,180,91,.15)","stroke-width":.6}, g);
```

改為：

```js
    el("path",{d:`M${p1} L${p2} A${R_WOUT},${R_WOUT} 0 0 1 ${p3} L${p4} A${R_WIN},${R_WIN} 0 0 0 ${p1} Z`,
      fill:i%2===0?"var(--yang)":"var(--yin)",opacity:.16,
      stroke:"rgba(217,180,91,.4)","stroke-width":1.2}, g);
```

- [ ] **Step 3: 地支文字不透明度 .9 → 1**

`base()` 內（~lines 254-257）：

```js
    el("text",{x:lx,y:ly,"text-anchor":"middle","dominant-baseline":"central",
      fill:i%2===0?"var(--yang)":"var(--yin)","font-size":17,
      "font-family":"'Noto Serif TC',serif","font-weight":600,opacity:.9},g)
```

`opacity:.9` 改為 `opacity:1`。

- [ ] **Step 4: 瀏覽器驗證**

Run: `python3 -m http.server 8000`（背景），開 `http://localhost:8000`。
Expected: 十二宮陰（藍）陽（橘）底色分明、宮界金線清楚；連線與星曜仍為視覺主角；console 無錯誤。截圖比對前後對比。

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "style: strengthen palace color contrast for screenshots"
```

---

