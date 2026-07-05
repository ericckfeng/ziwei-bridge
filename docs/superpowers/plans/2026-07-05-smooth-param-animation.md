# 平滑參數動畫與宮位對比強化 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 讓五個輸入參數（局數/年干/年支/生月/生時）皆可自動循環且變動時平滑動畫過渡，並強化十二宮底色對比以利截圖。

**Architecture:** 全部改動集中於單一 `index.html`。吉煞星層由「清空重畫」改為「狀態驅動」：以 `auxRenderState()` 計算每顆星的最終渲染角度（度數），參數改變時用 rAF 沿最短圓弧補間（與現有 `goTo()` 同款 easing）。四化徽章在年干改變時以獨立飛行層 `gFly` 從舊星飛向新星。循環控制透過「循環對象」選單泛化既有的 ◀▶／自動運行。

**Tech Stack:** 純 vanilla JS + SVG，零依賴，無 build 工具。

**Spec:** `docs/superpowers/specs/2026-07-05-smooth-param-animation-design.md`

## Global Constraints

- 全部程式碼在 `index.html` 單檔內（inline CSS + IIFE script），不得引入外部函式庫。
- UI 文字為繁體中文（zh-TW）。
- 本專案無測試框架；每個 task 的驗證為瀏覽器手動實測（`python3 -m http.server 8000` 後開 `http://localhost:8000`），並檢查 DevTools console 無錯誤。
- 動畫時長 700ms、easing `1-(1-k)^3`、`prefers-reduced-motion: reduce` 時直接落位——與現有 `goTo()` 一致。
- 局數旋轉時吉煞星必須保持固定不動（吉煞錨定出生四柱，這是現有的正確行為）。
- commit message 用英文 conventional commits。

**行號基準：** 以下行號以 commit `9a60b4a` 時的 `index.html` 為準；後面的 task 執行後行號會漂移，請以程式碼內容搜尋定位。

---

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

### Task 2: 吉煞星層改為狀態驅動（重構，行為不變）

**Files:**
- Modify: `index.html`（取代 `drawAux()` ~lines 368-402；修改 change listeners ~line 469、~lines 472-474；修改初始呼叫 ~line 478）

**Interfaces:**
- Consumes: 既有 `auxPos(gan,zhi,mon,hour)`、`pt()`、`el()`、`huaMap()`、`HUA_COLOR`、`HUA_NAME`、`R_AUX`。
- Produces:
  - `ptDeg(deg, r)` → `[x, y]`：絕對度數（0=子、每宮30°）轉座標。
  - `auxRenderState(gan, zhi, mon, hour)` → `{星名: {deg:Number, k:"ji"|"sha"}}`：13 顆吉煞星的最終渲染角度（含同宮散開偏移）。
  - `drawAux(state)`：以狀態表渲染 `gAux`（取代原本自行讀 select 的版本）。
  - `currentParams()` → `[gan字串, zhi數, mon數, hour數]`。
  - 全域 `let auxState`：目前畫面上的吉煞狀態表。
  - `onParamChange()`：年干/年支/生月/生時變更的統一處理函式（本 task 為即時落位版，Task 3 改為動畫版）。

- [ ] **Step 1: 以下列程式碼整段取代原 `drawAux()`（~lines 368-402）**

```js
/* 吉煞層 (不隨局數旋轉)：狀態驅動 */
function ptDeg(deg, r){
  const t = (180 + deg) * Math.PI/180;
  return [CX + r*Math.sin(t), CY - r*Math.cos(t)];
}
const SPREAD = [0,-8,8,-16,16,-24,24];
function auxRenderState(gan, zhi, mon, hour){
  const ap = auxPos(gan, zhi, mon, hour);
  const byBranch = {};
  for(const name in ap) (byBranch[ap[name].p] = byBranch[ap[name].p]||[]).push(name);
  const st = {};
  for(const br in byBranch)
    byBranch[br].forEach((name,j)=>{ st[name] = {deg: 30*(+br) + (SPREAD[j]||0), k: ap[name].k}; });
  return st;
}
let auxState = null;
function drawAux(state){
  gAux.innerHTML = "";
  const showJi = document.getElementById("tgJi").checked;
  const showSha = document.getElementById("tgSha").checked;
  if(!showJi && !showSha) return;
  const hm = document.getElementById("tgHua").checked ? huaMap() : {};
  for(const name in state){
    const {deg, k} = state[name];
    if(k==="ji" && !showJi) continue;
    if(k==="sha" && !showSha) continue;
    const [x,y] = ptDeg(deg, R_AUX);
    const col = k==="ji" ? "var(--ji)" : "var(--sha)";
    const mark = k==="ji" ? "◆" : "▲";
    const t = el("text",{x,y,"text-anchor":"middle","dominant-baseline":"central",
      fill:col,"font-size":11.5,"font-family":"'Noto Serif TC',serif","font-weight":600,
      "paint-order":"stroke",stroke:"var(--ink)","stroke-width":2.5}, gAux);
    t.textContent = mark + name;
    if(hm[name]!==undefined){
      const sp = document.createElementNS(NS,"tspan");
      sp.setAttribute("fill",HUA_COLOR[hm[name]]);
      sp.setAttribute("font-size","10");
      sp.textContent = "·"+HUA_NAME[hm[name]];
      t.appendChild(sp);
    }
  }
}
function currentParams(){ return [GAN[+ganSel.value], +zhiSel.value, +monSel.value, +hourSel.value]; }
function onParamChange(){
  auxState = auxRenderState(...currentParams());
  drawAux(auxState);
  draw(current,true);
  renderPanel(current);
}
```

注意：原 `drawAux` 內的 `SPREAD` 常數已上移為模組層常數，勿重複宣告。

- [ ] **Step 2: 更新事件監聽（~line 469 與 ~lines 472-474）**

原：

```js
[ganSel,zhiSel,monSel,hourSel].forEach(s=>s.addEventListener("change",()=>{drawAux();draw(current,true);renderPanel(current);}));
```

改為：

```js
[ganSel,zhiSel,monSel,hourSel].forEach(s=>s.addEventListener("change",onParamChange));
```

原：

```js
["tgJi","tgSha"].forEach(id=>
  document.getElementById(id).addEventListener("change",drawAux));
document.getElementById("tgHua").addEventListener("change",()=>{draw(current,true);drawAux();});
```

改為：

```js
["tgJi","tgSha"].forEach(id=>
  document.getElementById(id).addEventListener("change",()=>drawAux(auxState)));
document.getElementById("tgHua").addEventListener("change",()=>{draw(current,true);drawAux(auxState);});
```

- [ ] **Step 3: 更新初始化（~line 478）**

原：

```js
drawAxes(); drawAux(); draw(0,true); renderPanel(0);
```

改為：

```js
auxState = auxRenderState(...currentParams());
drawAxes(); drawAux(auxState); draw(0,true); renderPanel(0);
```

- [ ] **Step 4: 瀏覽器驗證（行為應與改動前完全相同）**

開 `http://localhost:8000`：
1. 初始畫面 13 顆吉煞星位置與改動前一致（甲年子支1月子時：祿存◆在寅、擎羊▲在卯、陀羅▲在丑…）。
2. 逐一改變年干/年支/生月/生時，吉煞星立即跳至新位置（本 task 尚無動畫，屬預期）。
3. 勾選框「六吉+祿存」「六煞」「四化」開關正常。
4. console 無錯誤。

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "refactor: make aux star layer state-driven for animation"
```

---

### Task 3: 參數變更的圓弧補間動畫

**Files:**
- Modify: `index.html`（取代 Task 2 的 `onParamChange()`）

**Interfaces:**
- Consumes: Task 2 的 `auxRenderState`、`drawAux(state)`、`currentParams`、`auxState`；既有 `reduceMotion`、`draw`、`renderPanel`、`current`。
- Produces: 全域 `let paramAnimId`（進行中的 rAF id，供重入取消）；動畫版 `onParamChange()`（簽名不變，Task 4/5 沿用）。

- [ ] **Step 1: 以動畫版取代 `onParamChange()`**

```js
let paramAnimId = null;
function onParamChange(){
  const target = auxRenderState(...currentParams());
  if(paramAnimId){ cancelAnimationFrame(paramAnimId); paramAnimId = null; }
  draw(current,true);
  if(reduceMotion || !auxState){
    auxState = target; drawAux(auxState); renderPanel(current); return;
  }
  const tw = {};
  for(const name in target){
    const s = auxState[name].deg;
    let d = target[name].deg - s;
    while(d > 180) d -= 360;
    while(d < -180) d += 360;
    tw[name] = {s, d};
  }
  const dur = 700, t0 = performance.now();
  function step(now){
    const k = Math.min(1,(now-t0)/dur), e = 1-Math.pow(1-k,3);
    const st = {};
    for(const name in target) st[name] = {deg: tw[name].s + tw[name].d*e, k: target[name].k};
    auxState = st;
    drawAux(auxState);
    if(k<1) paramAnimId = requestAnimationFrame(step);
    else { auxState = target; paramAnimId = null; drawAux(auxState); renderPanel(current); }
  }
  paramAnimId = requestAnimationFrame(step);
}
```

設計要點（實作時勿省略）：
- 最短圓弧：`d` 正規化到 (-180, 180]，星曜永遠走短邊。
- 重入：每影格把插值寫回 `auxState`，因此動畫中再次變更參數時，新補間自然從當前位置出發。
- `renderPanel` 只在動畫結束時呼叫一次（側欄不逐影格更新）。

- [ ] **Step 2: 瀏覽器驗證**

開 `http://localhost:8000`：
1. 生時 子→丑：文昌沿圓弧滑一宮、文曲反向滑一宮、空劫火鈴同步滑行，700ms 內完成，無黑閃。
2. 年干 甲→乙：祿存從寅滑到卯、羊陀跟著滑、魁鉞從丑未滑到子申。
3. 年支 子→丑（火鈴跳基點）：火鈴沿最短弧滑行。
4. 動畫進行中立刻再改另一參數：從當前位置平滑續接、不跳動。
5. 系統開啟「減少動態效果」（macOS: 系統設定→輔助使用→顯示器→減少動態效果）後重新整理：改參數直接落位。
6. 局數旋轉時吉煞星仍固定不動。
7. console 無錯誤。

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: animate aux stars along arc on parameter change"
```

---

### Task 4: 四化徽章飛行（年干變更）

**Files:**
- Modify: `index.html`（`gAux` 宣告處後新增 `gFly` ~line 274；`draw()` 與 `drawAux()` 的 `hm` 行加 flight 抑制；`onParamChange()` 加入飛行邏輯）

**Interfaces:**
- Consumes: Task 3 的 `onParamChange` 骨架；既有 `SIHUA`、`GAN`、`HUA_COLOR`、`HUA_NAME`、`starPos`、`pt`、`radiusOf`、`ZW_OFF`、`TF_OFF`、`R_AUX`；Task 2 的 `ptDeg`、`auxState`。
- Produces: 全域 `gFly`（最上層 SVG group）、`let huaFlight`（布林，飛行中抑制附著式四化）、`let lastGan`（上一次年干字串）、`starCoordFor(name, tableIdx, auxSt)`、`renderFly(flights, e)`。

- [ ] **Step 1: 新增 `gFly` 圖層**

在（~line 274）：

```js
const gAux = el("g",{});
```

之後加一行：

```js
const gFly = el("g",{});
```

（`el` 依序 append，`gFly` 在最後＝最上層。）

- [ ] **Step 2: `draw()` 與 `drawAux()` 的四化抑制**

`draw()` 內（~line 333）：

```js
  const hm = document.getElementById("tgHua").checked ? huaMap() : {};
```

改為：

```js
  const hm = (!huaFlight && document.getElementById("tgHua").checked) ? huaMap() : {};
```

`drawAux()` 內同一行同樣修改（兩處都要改）。

- [ ] **Step 3: 新增飛行輔助函式（放在 `onParamChange` 之前）**

```js
let huaFlight = false, lastGan = "甲";
function starCoordFor(name, tableIdx, auxSt){
  if(ZW_OFF.hasOwnProperty(name) || TF_OFF.hasOwnProperty(name)){
    const p = starPos(tableIdx);
    return pt(p[name], radiusOf(name));
  }
  return ptDeg(auxSt[name].deg, R_AUX);
}
function renderFly(flights, e){
  gFly.innerHTML = "";
  flights.forEach(f=>{
    const x = f.from[0] + (f.to[0]-f.from[0])*e;
    const y = f.from[1] + (f.to[1]-f.from[1])*e;
    el("circle",{cx:x,cy:y,r:8.5,fill:HUA_COLOR[f.i],stroke:"var(--ink)","stroke-width":1.2}, gFly);
    el("text",{x,y,"text-anchor":"middle","dominant-baseline":"central",
      fill:"#fff","font-size":10.5}, gFly).textContent = HUA_NAME[f.i];
  });
}
```

- [ ] **Step 4: 以含飛行的版本取代 `onParamChange()`**

```js
function onParamChange(){
  const newGan = GAN[+ganSel.value];
  const target = auxRenderState(...currentParams());
  const ganChanged = newGan !== lastGan;
  if(paramAnimId){ cancelAnimationFrame(paramAnimId); paramAnimId = null; }
  huaFlight = false; gFly.innerHTML = "";
  if(reduceMotion || !auxState){
    auxState = target; lastGan = newGan;
    drawAux(auxState); draw(current,true); renderPanel(current); return;
  }
  let flights = null;
  if(ganChanged && document.getElementById("tgHua").checked){
    const oldStars = SIHUA[lastGan], newStars = SIHUA[newGan];
    flights = [0,1,2,3].map(i=>({
      i,
      from: starCoordFor(oldStars[i], current, auxState),
      to:   starCoordFor(newStars[i], current, target)
    }));
    huaFlight = true;
  }
  lastGan = newGan;
  draw(current,true);
  const tw = {};
  for(const name in target){
    const s = auxState[name].deg;
    let d = target[name].deg - s;
    while(d > 180) d -= 360;
    while(d < -180) d += 360;
    tw[name] = {s, d};
  }
  const dur = 700, t0 = performance.now();
  function step(now){
    const k = Math.min(1,(now-t0)/dur), e = 1-Math.pow(1-k,3);
    const st = {};
    for(const name in target) st[name] = {deg: tw[name].s + tw[name].d*e, k: target[name].k};
    auxState = st;
    drawAux(auxState);
    if(flights) renderFly(flights, e);
    if(k<1) paramAnimId = requestAnimationFrame(step);
    else {
      auxState = target; paramAnimId = null;
      huaFlight = false; gFly.innerHTML = "";
      drawAux(auxState); draw(current,true); renderPanel(current);
    }
  }
  paramAnimId = requestAnimationFrame(step);
}
```

設計要點：
- `from` 用**變更前**的 `auxState` 與 `SIHUA[lastGan]`，`to` 用 `target` 與新干——順序不可顛倒（`lastGan = newGan` 必須在 flights 計算之後）。
- 飛行期間 `huaFlight=true`：主星的四化光環/tspan 與吉煞星的四化 tspan 都暫時隱藏，只看得到 4 顆飛行徽章；結束後恢復附著樣式。
- 已知限制：飛行中再次變更年干，新的 `from` 取自上一目標的附著位置而非飛行中座標，會有一次小跳點，可接受。

- [ ] **Step 5: 瀏覽器驗證**

開 `http://localhost:8000`：
1. 年干 甲→乙：祿權科忌 4 顆圓徽章分別從 廉貞/破軍/武曲/太陽 飛向 天機/天梁/紫微/太陰，落地後恢復「星名·祿」樣式與主星光環。
2. 年干 乙→丙：科徽章飛向吉煞環上的文昌（跨環飛行正常）。
3. 關掉「四化」勾選框再改年干：無飛行、無錯誤。
4. 改年支/生月/生時（年干不變）：無飛行徽章出現，吉煞星照常滑行。
5. reduced-motion 下改年干：直接落位。
6. console 無錯誤。

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: fly sihua badges between stars on year-stem change"
```

---

### Task 5: 循環對象選單與自動運行泛化

**Files:**
- Modify: `index.html`（控制列 HTML ~lines 89-95；「使用方法」卡片 ~lines 129-133；JS 控制元件區 ~lines 449-468）

**Interfaces:**
- Consumes: Task 3/4 的 `onParamChange()`；既有 `goTo`、`schedule`、`playing`、`playTimer`、`speed`、`current`。
- Produces: `cycleSel`（select 元素）、`CYCLE`（循環對象設定表）、`stepCycle(dir)`。

- [ ] **Step 1: 控制列加入「循環」選單**

原（~lines 89-91）：

```html
    <div class="ctlrow sep">
      <label>局數</label><select id="tableSel"></select>
      <button id="prevBtn">◀</button><button id="nextBtn">▶</button>
```

改為：

```html
    <div class="ctlrow sep">
      <label>局數</label><select id="tableSel"></select>
      <label>循環</label><select id="cycleSel"></select>
      <button id="prevBtn">◀</button><button id="nextBtn">▶</button>
```

- [ ] **Step 2: 建立 `CYCLE` 設定與 `stepCycle`**

在控制元件 const 宣告區（~line 450 的 `const tableSel=...` 區塊）加入 `cycleSel`：

```js
const cycleSel = document.getElementById("cycleSel");
```

在各選單填充程式碼（~lines 453-457）之後加入：

```js
const CYCLE = [
  {label:"局數"},
  {label:"年干", sel:ganSel,  n:10, min:0},
  {label:"年支", sel:zhiSel,  n:12, min:0},
  {label:"生月", sel:monSel,  n:12, min:1},
  {label:"生時", sel:hourSel, n:12, min:0}
];
CYCLE.forEach((c,i)=>{ const o=document.createElement("option"); o.value=i; o.textContent=c.label; cycleSel.appendChild(o); });
function stepCycle(dir){
  const c = CYCLE[+cycleSel.value];
  if(!c.sel){ goTo(current+dir); return; }
  const v = +c.sel.value - c.min;
  c.sel.value = c.min + ((v + dir) % c.n + c.n) % c.n;
  onParamChange();
}
```

（`CYCLE[0]` 無 `sel` ＝ 局數，走既有 `goTo`。程式化設值不會觸發 change 事件，故需手動呼叫 `onParamChange()`。）

- [ ] **Step 3: ◀▶ 與自動運行改用 `stepCycle`**

原（~lines 460-461）：

```js
document.getElementById("prevBtn").addEventListener("click",()=>goTo(current-1));
document.getElementById("nextBtn").addEventListener("click",()=>goTo(current+1));
```

改為：

```js
document.getElementById("prevBtn").addEventListener("click",()=>stepCycle(-1));
document.getElementById("nextBtn").addEventListener("click",()=>stepCycle(1));
```

原 `schedule()`（~lines 443-447）：

```js
function schedule(){
  clearTimeout(playTimer);
  if(!playing) return;
  playTimer = setTimeout(()=>{ goTo(current+1); schedule(); }, +speed.value*1000);
}
```

改為：

```js
function schedule(){
  clearTimeout(playTimer);
  if(!playing) return;
  playTimer = setTimeout(()=>{ stepCycle(1); schedule(); }, +speed.value*1000);
}
```

（`schedule` 是 function declaration、`stepCycle` 於其後定義，hoisting 下呼叫時已存在，無 TDZ 問題。）

- [ ] **Step 4: 更新「使用方法」卡片**

原第一條（~line 130）：

```html
        <li>選「局數」跳至任一表，或按「自動運行」由表一至表十二循環旋轉。</li>
```

改為：

```html
        <li>「循環」選單決定 ◀▶ 與「自動運行」作用的參數（局數／年干／年支／生月／生時）；按「自動運行」即以停留秒數自動循環該參數，所有變化皆以動畫平滑過渡。</li>
```

- [ ] **Step 5: 瀏覽器驗證（完整回歸）**

開 `http://localhost:8000`：
1. 循環＝局數：◀▶ 與自動運行行為與從前相同（主盤旋轉、吉煞不動）。
2. 循環＝年干：自動運行甲→乙→…→癸→甲 循環，每步祿存羊陀魁鉞滑行、四化徽章飛行。
3. 循環＝年支／生月／生時：各自循環到底回頭（生月 12月→1月），星曜滑行無黑閃。
4. 停留秒數滑桿對所有循環對象生效；「⏸ 停止運行」可停。
5. 循環進行中手動改另一個參數：平滑續接。
6. 直接用下拉選單改局數/年干等，行為正常（change 事件路徑）。
7. console 無錯誤。

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: add cycle target selector for auto-run of all inputs"
```

---

## 完成後

依 superpowers:finishing-a-development-branch 處理（本專案直接在 main 工作則跳過分支整併），並由使用者做最終截圖驗收（宮位對比、動畫可觀察性）。
