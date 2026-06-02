# Pulso 2026 — Redesign Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Redesign `index.html` with a light Editorial Claro theme using Colombia flag colors, candidate photos, a dismissible instructions banner, simplified sliders, corrected math, and Colombian Spanish terminology throughout.

**Architecture:** Single-file HTML project. All changes target `index.html`. Tasks are organized as sequential edits to the `<style>` block (lines 32–174), the HTML body (lines 176–270), and the `<script>` block (lines 271–485). JavaScript logic is preserved; only presentation and one denominator calculation change.

**Tech Stack:** Vanilla HTML/CSS/JS · Google Fonts (existing) · Wikimedia Commons (candidate photos) · `localStorage` (banner dismiss).

---

## Chunk 1: Math & Code Fixes

> Fix the four issues identified in code review. No visual changes. Verify each fix in the browser console before moving on.

---

### Task 1: Remove dead code — `ANCHOR` and `resolve()`

**Files:**
- Modify: `index.html:305-306`

- [ ] **Step 1: Delete lines 305–306**

Remove these two lines entirely from the `<script>` block:

```javascript
const ANCHOR = modelFor(PRESETS.base.__resolved||resolve(PRESETS.base)).pa;
function resolve(p){const c={retA:p.retA,retC:p.retC,machine:p.machine};BLOCS.forEach(b=>c[b.id]={abst:p[b.id][0],share:p[b.id][1]});return c;}
```

Nothing else references `ANCHOR` or `resolve`, so they can be deleted outright.

- [ ] **Step 2: Verify no console errors**

Open `index.html` in browser → DevTools Console. Should be zero errors. Move a slider to confirm the model still runs.

---

### Task 2: Fix `modelFor()` — include blank votes in `pa`/`pc` denominator

**Files:**
- Modify: `index.html:300-302` (line 303 is the closing `}` of `modelFor()` — do NOT delete it)

- [ ] **Step 1: Replace lines 300–302 inside `modelFor()` (leave line 303 `}` untouched)**

Old:
```javascript
  const total=abe+cep;
  let pa=abe/total*100; pa=Math.max(0,Math.min(100,pa+s.machine)); const pc=100-pa;
  return {pa,pc,abeV:total*pa/100,cepV:total*pc/100,total,blancos,turnout:(total+blancos)/CENSO*100};
```

New:
```javascript
  const candTotal=abe+cep;
  let abeShare=abe/candTotal*100;
  abeShare=Math.max(0,Math.min(100,abeShare+s.machine));
  const abeV=candTotal*abeShare/100, cepV=candTotal*(100-abeShare)/100;
  const grandTotal=candTotal+blancos;
  const pa=abeV/grandTotal*100, pc=cepV/grandTotal*100;
  return {pa,pc,abeV,cepV,total:candTotal,blancos,turnout:grandTotal/CENSO*100};
```

**Why this is correct:**
- Machine still adjusts the abe/cep candidate split (not the valid-vote share directly).
- `pa` and `pc` are now shares of all valid votes (candidates + blank), so `pa + pc < 100`.
- `abeV` and `cepV` (absolute vote counts) are unchanged in meaning.
- The bar will show a tiny gap at the right end representing blank votes — this is intentional and honest.

- [ ] **Step 2: Verify new denominator in console**

With "Caso base" applied, run in browser console:
```javascript
const m = modelFor(cloneState());
console.log('pa:', m.pa.toFixed(2), 'pc:', m.pc.toFixed(2), 'sum:', (m.pa+m.pc).toFixed(2));
```
Expected: `pa: ~52.42  pc: ~46.73  sum: ~99.15` (blank takes the remaining ~0.85%).

---

### Task 3: Fix Monte Carlo win condition

**Files:**
- Modify: `index.html:382`

- [ ] **Step 1: Update `monteCarlo` win test**

Old (line 382):
```javascript
    const pa=modelFor(s).pa; if(pa>50)wins++;
```

New:
```javascript
    const {pa,pc}=modelFor(s); if(pa>pc)wins++;
```

`pa > pc` is equivalent to `abe > cep` regardless of how many blank votes exist. The old `pa > 50` was correct when `pa + pc = 100` but breaks once blancos are in the denominator (both candidates can be below 50%).

- [ ] **Step 2: Verify histogram still runs**

Reload browser. After a few seconds the probability histogram should populate. No console errors.

---

### Task 4: Fix histogram axis — position "50%" at true 33% from left

The 20-bin histogram covers pa values 45%–60% (range = 15pp). The "50%" mark sits at (50−45)/15 = **33.3%** from the left, not at the visual center (50%).

**Files:**
- Modify: `index.html` — `.hist-axis` CSS rule + histogram axis HTML

- [ ] **Step 1: Replace `.hist-axis` CSS rule**

Old (inside `<style>`):
```css
.hist-axis{display:flex;justify-content:space-between;font-family:"JetBrains Mono",monospace;
  font-size:9.5px;color:var(--muted);margin-top:4px}
```

New:
```css
.hist-axis{position:relative;height:16px;margin-top:4px;font-family:"JetBrains Mono",monospace;font-size:9.5px;color:var(--muted)}
.hist-axis span{position:absolute;white-space:nowrap}
```

- [ ] **Step 2: Replace histogram axis HTML with 5 correctly-positioned labels**

The 20-bin range is 45–60% (15pp total). Label positions as % of axis width:
- 45% → 0%
- 47.5% → (2.5/15) = 16.7%
- 50% → (5/15) = 33.3%
- 55% → (10/15) = 66.7%
- 60%+ → 100%

Old (line 218):
```html
<div class="hist-axis"><span>45% Abe</span><span>50%</span><span>60%+</span></div>
```

New:
```html
<div class="hist-axis">
  <span style="left:0">45%</span>
  <span style="left:16.7%;transform:translateX(-50%)">47.5%</span>
  <span style="left:33.3%;transform:translateX(-50%)">50%</span>
  <span style="left:66.7%;transform:translateX(-50%)">55%</span>
  <span style="right:0">60%+</span>
</div>
```

- [ ] **Step 3: Verify visually**

Open browser. The "50%" label should appear roughly one-third from the left of the histogram bars — not at center. Five labels should be visible and evenly spaced by probability value.

**Note:** The `.bar-labels` row below the vote bar also has a center "50%" label — that one IS correctly at the visual center because the bar runs 0–100%, not 45–60%. Do not apply the left:33.3% fix to `.bar-labels`.

---

### Task 5: Update `recompute()` blank-pct calc + commit

**Files:**
- Modify: `index.html:417`

- [ ] **Step 1: Simplify `blkPct` derivation**

Old (line 417):
```javascript
  const blkPct=(m.blancos/(m.total+m.blancos)*100);
```

New (equivalent, but derived from already-correct pa/pc):
```javascript
  const blkPct=100-m.pa-m.pc;
```

- [ ] **Step 2: Verify margin display**

Reload. Apply "Caso base". The margin text should show blank vote % of ~0.9%.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "fix: include blancos in pa/pc denominator, remove dead code, fix histogram axis, fix MC win condition"
```

---

## Chunk 2: CSS Redesign

> Replace the entire color palette and add new component styles. No HTML or JS changes yet.

---

### Task 6: Replace `:root` variables and `body` background

**Files:**
- Modify: `index.html:33-46`

- [ ] **Step 1: Replace `:root` block (lines 33–37)**

Old:
```css
:root{
  --bg:#15110C; --panel:#1F1A13; --panel-2:#262017; --line:#3A3125;
  --cream:#F1E8D8; --muted:#A89578;
  --abe:#E2922E; --abe-soft:#3a2c16; --cep:#34B6A4; --cep-soft:#13322e; --warn:#D8643C;
}
```

New:
```css
:root{
  --bg:#F5F3EF; --panel:#FFFFFF; --panel-2:#F0EEE9; --line:#E0DDD6;
  --cream:#1a1a1a; --muted:#88887A;
  --abe:#003893; --abe-soft:#EEF2FF; --cep:#CE1126; --cep-soft:#FFF5F5; --warn:#CE1126;
  --gold:#FCD116;
}
```

- [ ] **Step 2: Replace `body` block (lines 39–46)**

Old:
```css
body{
  background:
    radial-gradient(1200px 600px at 12% -10%, #2a2013 0%, transparent 55%),
    radial-gradient(1000px 700px at 112% 8%, #11302c 0%, transparent 50%),
    var(--bg);
  color:var(--cream); font-family:"Libre Franklin",system-ui,sans-serif; line-height:1.5;
  -webkit-font-smoothing:antialiased; min-height:100vh;
}
```

New:
```css
body{
  background:var(--bg);
  color:var(--cream); font-family:"Libre Franklin",system-ui,sans-serif; line-height:1.5;
  -webkit-font-smoothing:antialiased; min-height:100vh;
}
```

---

### Task 7: Update typographic, stamp, button, and preset styles

**Files:**
- Modify: `index.html` (`<style>` block)

- [ ] **Step 1: Update `h1 em` color**

Old: `h1 em{font-style:italic;color:var(--abe)}`
New: `h1 em{font-style:italic;color:var(--cep)}`

- [ ] **Step 2: Update `.stamp`**

Old:
```css
.stamp{...color:var(--warn);...border:1px solid #5a3a2a;background:#2a1a12;...}
```
New:
```css
.stamp{font-family:"JetBrains Mono",monospace;font-size:10.5px;letter-spacing:.04em;
  color:var(--cep);margin-top:10px;display:inline-block;
  border:1px solid #f0c0c0;background:#fff8f8;padding:5px 10px;border-radius:7px}
```

- [ ] **Step 3: Update `.btn` and `.btn.share`**

Old:
```css
.btn{...background:var(--panel);color:var(--cream);border:1px solid var(--line);...}
.btn.share{background:var(--abe);color:#1a1206;...}
```
New:
```css
.btn{font-family:"JetBrains Mono",monospace;font-size:11px;letter-spacing:.04em;
  background:var(--panel);color:#333;border:1px solid var(--line);
  padding:9px 14px;border-radius:999px;cursor:pointer;transition:.18s}
.btn:hover{border-color:var(--abe)}
.btn.share{background:var(--cep);color:#fff;border-color:var(--cep);font-weight:700}
```

- [ ] **Step 4: Update `.presets button` and `.presets button.on`**

```css
.presets button{font-family:"JetBrains Mono",monospace;font-size:11px;letter-spacing:.05em;
  background:var(--panel);color:var(--muted);border:1px solid var(--line);
  padding:9px 13px;border-radius:999px;cursor:pointer;transition:.18s}
.presets button:hover{color:var(--cream);border-color:var(--abe)}
.presets button.on{background:var(--abe);color:#fff;border-color:var(--abe);font-weight:700}
```

- [ ] **Step 5: Update `.note`**

Old: `background:var(--abe-soft);border:1px solid #5a4012;...color:#e8d3a8;`
New:
```css
.note{background:var(--abe-soft);border:1px solid #c8d4f8;border-radius:10px;padding:12px 14px;font-size:12px;color:#333;margin-top:14px}
```

- [ ] **Step 6: Update `.toast`**

Old: `background:var(--abe);color:#1a1206;`
New: `background:var(--abe);color:#fff;` (keep abe for toast, blue on white)

- [ ] **Step 7: Update `input[type=range]` thumb and track**

```css
input[type=range]{-webkit-appearance:none;appearance:none;width:100%;height:6px;border-radius:6px;
  background:#e8e8e4;outline:none;cursor:pointer;border:1px solid var(--line)}
input[type=range].split{background:linear-gradient(90deg,var(--cep-soft),var(--abe-soft))}
input[type=range].blank{background:linear-gradient(90deg,#f0ede8,#e0ddd6)}
input[type=range]::-webkit-slider-thumb{-webkit-appearance:none;width:20px;height:20px;border-radius:50%;
  background:#fff;border:3px solid #333;box-shadow:0 2px 6px rgba(0,0,0,.2)}
input[type=range]::-moz-range-thumb{width:18px;height:18px;border-radius:50%;background:#fff;border:3px solid #333}
.val.blk{color:#7a6a50}
```

- [ ] **Step 8: Update sticky bar for light theme**

```css
.sticky-bar{...background:rgba(245,243,239,.95);backdrop-filter:blur(14px);-webkit-backdrop-filter:blur(14px);
  border-bottom:1px solid var(--line);...}
.sb-a{color:var(--abe);font-weight:700}
.sb-c{color:var(--cep);font-weight:700}
.sb-vs{color:var(--line);font-size:11px}
.sb-prob{color:var(--muted);font-size:11px;text-align:right}
```

- [ ] **Step 9: Update `.card` for light theme (used until HTML restructure)**

```css
.card{background:linear-gradient(180deg,var(--panel-2),var(--panel));
  border:1px solid var(--line);border-radius:18px;padding:22px;
  box-shadow:0 4px 24px rgba(0,0,0,.06);animation:rise .7s .05s ease both}
```

- [ ] **Step 10: Update `.bar` for light theme**

```css
.bar{height:28px;border-radius:8px;overflow:hidden;display:flex;background:var(--line);border:1px solid var(--line)}
.bar .a{background:linear-gradient(90deg,#002070,var(--abe));transition:width .5s cubic-bezier(.4,0,.1,1)}
.bar .c{background:linear-gradient(90deg,var(--cep),#a00d1e);transition:width .5s cubic-bezier(.4,0,.1,1)}
```

---

### Task 8: Add new CSS components + commit

Add these new CSS rules at the end of the `<style>` block, before `</style>`.

**Files:**
- Modify: `index.html` (end of `<style>` block)

- [ ] **Step 1: Add flag stripe rules**

```css
/* Colombia flag stripes */
.flag-stripe{height:6px;background:linear-gradient(90deg,var(--gold) 50%,var(--abe) 50% 75%,var(--cep) 75%)}
.flag-footer{height:4px;background:linear-gradient(90deg,var(--gold) 50%,var(--abe) 50% 75%,var(--cep) 75%);border-radius:2px;margin:30px 0 20px;opacity:.5}
```

- [ ] **Step 2: Add candidate hero rules**

```css
/* candidate hero */
.hero{display:grid;grid-template-columns:1fr auto 1fr;gap:14px;align-items:center;margin:20px 0 14px}
.candidate{background:var(--panel);border:2px solid var(--line);border-radius:16px;padding:18px;display:flex;align-items:center;gap:14px;box-shadow:0 2px 12px rgba(0,0,0,.06)}
.candidate.abe{border-color:var(--abe);background:linear-gradient(135deg,var(--abe-soft),var(--panel))}
.candidate.cep{border-color:var(--cep);background:linear-gradient(135deg,var(--cep-soft),var(--panel))}
.candidate img{width:64px;height:64px;border-radius:50%;object-fit:cover;object-position:top;border:3px solid var(--panel);box-shadow:0 2px 8px rgba(0,0,0,.15);flex-shrink:0}
.candidate.abe img{border-color:var(--abe)} .candidate.cep img{border-color:var(--cep)}
.candidate-info{flex:1;min-width:0}
.cand-name{font-size:13px;font-weight:600;color:var(--muted);line-height:1.3;margin-bottom:2px}
.cand-pct{font-family:"Fraunces",serif;font-weight:900;font-size:36px;line-height:1}
.abe .cand-pct{color:var(--abe)} .cep .cand-pct{color:var(--cep)}
.cand-votes{font-size:11px;color:var(--muted);font-family:"JetBrains Mono",monospace}
.vs-badge{font-family:"Fraunces",serif;font-weight:900;font-size:18px;color:var(--line);text-align:center;flex-shrink:0}
.bar-labels{display:flex;justify-content:space-between;font-family:"JetBrains Mono",monospace;font-size:10px;color:var(--muted);margin-top:4px}
@media(max-width:560px){.hero{grid-template-columns:1fr}.vs-badge{display:none}.candidate{flex-direction:column;text-align:center}}
```

- [ ] **Step 3: Add how-to banner rules**

```css
/* how-to banner */
.how-to{background:var(--panel);border:1.5px solid var(--abe);border-radius:14px;padding:16px 20px;margin-bottom:20px;display:flex;gap:16px;align-items:flex-start}
.how-to-icon{font-size:22px;flex-shrink:0;margin-top:2px}
.how-to h4{font-size:13px;font-weight:700;color:var(--abe);margin-bottom:6px}
.how-to ol{padding-left:16px}
.how-to li{font-size:13px;color:#444;line-height:1.6;margin-bottom:2px}
.how-to li b{color:var(--cream)}
.how-to-dismiss{font-size:11px;color:var(--muted);margin-top:8px;cursor:pointer;display:inline-block;background:none;border:none;padding:0;font-family:inherit}
.how-to-dismiss:hover{color:var(--abe)}
```

- [ ] **Step 4: Add stat card and advanced accordion rules**

```css
/* stat cards */
.stat-card{background:var(--panel);border:1px solid var(--line);border-radius:16px;padding:20px;box-shadow:0 2px 12px rgba(0,0,0,.05)}
.stat-card h3{font-family:"JetBrains Mono",monospace;font-size:10px;letter-spacing:.18em;text-transform:uppercase;color:var(--muted);margin-bottom:12px}

/* advanced accordion */
.advanced-section{margin-top:10px;border-top:1px solid var(--line);padding-top:6px}
.advanced-toggle{text-align:center;font-size:12px;color:var(--muted);cursor:pointer;padding:8px;background:none;border:none;width:100%;font-family:inherit}
.advanced-toggle:hover{color:var(--abe)}
.advanced-body{overflow:hidden;transition:max-height .32s ease,opacity .22s;max-height:0;opacity:0;pointer-events:none}
.advanced-body.open{max-height:1000px;opacity:1;pointer-events:auto}
```

- [ ] **Step 5: Verify page still loads**

Open browser. The page should be mostly light now (background cream/white). Some layout may look off until HTML changes in Chunk 3 — that's expected. No console errors.

- [ ] **Step 6: Commit CSS changes**

```bash
git add index.html
git commit -m "style: Editorial Claro theme — Colombia flag colors, light cards, new component CSS"
```

---

## Chunk 3: HTML Structure

> Restructure the body: add flag stripes, replace the results grid with the candidate hero, add the how-to banner, simplify orphan sliders, and update all terminology.

---

### Task 9: Add flag stripe + update header kicker

**Files:**
- Modify: `index.html` (body — top of page and header)

- [ ] **Step 1: Add flag stripe as first child of `<body>`**

Immediately after `<body>`, before `<div class="sticky-bar"...>`:
```html
<div class="flag-stripe"></div>
```

- [ ] **Step 2: Update kicker text**

Old:
```html
<div class="kicker">Colombia · Balotaje 21 de junio 2026</div>
```
New:
```html
<div class="kicker">Colombia · Segunda vuelta · 21 de junio de 2026</div>
```

---

### Task 10: Add how-to banner after `</header>`

**Files:**
- Modify: `index.html` (inside `.wrap`, after `</header>`)

- [ ] **Step 1: Insert how-to banner HTML**

After the closing `</header>` tag (after the `.actions` div), add:
```html
<div class="how-to" id="howTo">
  <div class="how-to-icon">🗳️</div>
  <div>
    <h4>¿Cómo usar este modelo?</h4>
    <ol>
      <li><b>Mira el resultado base</b> — así quedaría la segunda vuelta según los supuestos más probables.</li>
      <li><b>Mueve las barras</b> en "¿A dónde van los votos libres?" para repartir los votos de Paloma, Fajardo, Claudia López y los demás entre los dos finalistas.</li>
      <li><b>Prueba los escenarios</b> preestablecidos para ver casos extremos: derecha unida, voto de castigo, abstención alta.</li>
      <li><b>Comparte</b> tu escenario favorito con el botón "Compartir este escenario".</li>
    </ol>
    <button class="how-to-dismiss" id="howToDismiss">Entendido, no volver a mostrar ✕</button>
  </div>
</div>
```

---

### Task 11: Replace results grid with candidate hero + stat cards

**Files:**
- Modify: `index.html:200-220`

The existing `<div class="grid" id="mainGrid">` contains two `.card` divs. Replace the entire block.

- [ ] **Step 1: Replace `<div class="grid" id="mainGrid">...</div>` (lines 200–220)**

Old:
```html
<div class="grid" id="mainGrid">
  <div class="card">
    <h3>Proyección nacional</h3>
    <div class="verdict" id="verdict">—</div>
    <div class="names">
      <div class="nm-a">Abelardo de la Espriella<b id="pa">—</b><span class="v" id="va"></span></div>
      <div class="nm-c">Iván Cepeda<b id="pc">—</b><span class="v" id="vc"></span></div>
    </div>
    <div class="fifty"><div class="tick"></div></div>
    <div class="bar"><div class="a" id="ba"></div><div class="c" id="bc"></div></div>
    <div class="margin" id="margin"></div>
  </div>

  <div class="card">
    <h3>Probabilidad de victoria</h3>
    <div class="prob-num" id="probNum">—</div>
    <div class="prob-lab" id="probLab"></div>
    <div class="hist" id="hist"></div>
    <div class="hist-axis"><span>45% Abe</span><span>50%</span><span>60%+</span></div>
  </div>
</div>
```

New (keep `id="mainGrid"` on the outer wrapper — the sticky bar IntersectionObserver depends on it):
```html
<div id="mainGrid">
  <!-- Candidate hero -->
  <div class="hero">
    <div class="candidate abe">
      <img src="https://upload.wikimedia.org/wikipedia/commons/thumb/6/69/Abelardo_de_la_Espriella_2025.jpg/200px-Abelardo_de_la_Espriella_2025.jpg"
           alt="Abelardo de la Espriella" onerror="this.style.display='none'">
      <div class="candidate-info">
        <div class="cand-name">Abelardo<br>de la Espriella</div>
        <div class="cand-pct" id="pa">—</div>
        <div class="cand-votes" id="va"></div>
      </div>
    </div>
    <div class="vs-badge">VS</div>
    <div class="candidate cep">
      <div class="candidate-info" style="text-align:right">
        <div class="cand-name">Iván<br>Cepeda</div>
        <div class="cand-pct" id="pc">—</div>
        <div class="cand-votes" id="vc"></div>
      </div>
      <img src="https://upload.wikimedia.org/wikipedia/commons/thumb/5/58/Perfil_Iv%C3%A1n_Cepeda_%28cropped%29.jpg/200px-Perfil_Iv%C3%A1n_Cepeda_%28cropped%29.jpg"
           alt="Iván Cepeda" onerror="this.style.display='none'">
    </div>
  </div>

  <!-- Vote bar -->
  <div class="fifty"><div class="tick"></div></div>
  <div class="bar"><div class="a" id="ba"></div><div class="c" id="bc"></div></div>
  <div class="bar-labels"><span>← más Cepeda</span><span>50%</span><span>más Abelardo →</span></div>
  <div class="verdict" id="verdict" style="margin-top:10px;text-align:center;font-family:'Fraunces',serif;font-style:italic;font-size:clamp(15px,2.5vw,19px);color:#444">—</div>

  <!-- Stat cards -->
  <div class="grid" style="margin-top:16px">
    <div class="stat-card">
      <h3>Probabilidad de victoria</h3>
      <div class="prob-num" id="probNum">—</div>
      <div class="prob-lab" id="probLab"></div>
      <div class="hist" id="hist"></div>
      <div class="hist-axis">
        <span style="left:0">45%</span>
        <span style="left:16.7%;transform:translateX(-50%)">47.5%</span>
        <span style="left:33.3%;transform:translateX(-50%)">50%</span>
        <span style="left:66.7%;transform:translateX(-50%)">55%</span>
        <span style="right:0">60%+</span>
      </div>
    </div>
    <div class="stat-card">
      <h3>Margen y participación</h3>
      <div class="margin" id="margin" style="font-family:'JetBrains Mono',monospace;font-size:11.5px;color:var(--muted);line-height:1.8"></div>
    </div>
  </div>
</div>
```

**Note:** All IDs used by `recompute()` and `paintProb()` are preserved (`pa`, `pc`, `va`, `vc`, `ba`, `bc`, `verdict`, `margin`, `probNum`, `probLab`, `hist`). The `animateVal()` calls and DOM lookups in the JS require no changes.

- [ ] **Step 2: Verify candidate hero renders**

Open browser. You should see two candidate cards side by side with photos, names, and initial "—" values. After a moment they should populate with numbers.

---

### Task 12: Simplify orphan vote sliders — split-only in main, advanced accordion

**Files:**
- Modify: `index.html:229` (section title) + `index.html:336-352` (bloc template in JS)

- [ ] **Step 1: Update section title and add subtitle**

Old:
```html
<div class="sec-title">Reparto de los votos huérfanos</div>
```
New:
```html
<div class="sec-title">¿A dónde van los votos libres?</div>
<p style="font-size:13px;color:var(--muted);margin-bottom:14px;max-width:66ch">Los candidatos que no pasaron sumaron <b style="color:var(--cream)">9.6M votos (15.4%)</b>. ¿A dónde irán el 21 de junio? Arrastra las barras.</p>
```

- [ ] **Step 2: Replace the bloc `innerHTML` template in the JS `BLOCS.forEach` (lines 336–352)**

Old template:
```javascript
  el.innerHTML=`
    <div class="bloc-head toggle">
      <div class="bloc-name">${b.name}<small>${b.desc}</small></div>
      <div style="display:flex;align-items:center;gap:8px;flex-shrink:0">
        <div class="pool">${b.pct.toFixed(1)}% · ${fmtM(b.pool)}</div>
        <span class="chev">▾</span>
      </div>
    </div>
    <div class="bloc-body">
      <div class="ctrl"><div class="lab"><span>Se queda en casa</span><span class="val" id="${b.id}-av">0%</span></div>
        <input type="range" id="${b.id}-abst" min="0" max="100" step="1" value="0"></div>
      <div class="ctrl"><div class="lab"><span>Vota en blanco</span><span class="val blk" id="${b.id}-bv">0%</span></div>
        <input type="range" class="blank" id="${b.id}-blank" min="0" max="60" step="1" value="0"></div>
      <div class="ctrl"><div class="lab"><span>De los que votan por alguno</span><span class="val" id="${b.id}-sv">0% Abe</span></div>
        <input type="range" class="split" id="${b.id}-share" min="0" max="100" step="1" value="0">
        <div class="ends"><span class="l">◄ Cepeda</span><span class="r">Abelardo ►</span></div></div>
    </div>`;
```

New template (split slider first, abstention/blank in collapsible advanced section):
```javascript
  el.innerHTML=`
    <div class="bloc-head toggle">
      <div class="bloc-name">${b.name}<small>${b.desc}</small></div>
      <div style="display:flex;align-items:center;gap:8px;flex-shrink:0">
        <div class="pool">${b.pct.toFixed(1)}% · ${fmtM(b.pool)}</div>
        <span class="chev">▾</span>
      </div>
    </div>
    <div class="bloc-body">
      <div class="ctrl">
        <div class="lab"><span>¿A quién van sus votos?</span><span class="val" id="${b.id}-sv">0% Abe</span></div>
        <input type="range" class="split" id="${b.id}-share" min="0" max="100" step="1" value="0">
        <div class="ends"><span class="l">◄ Cepeda</span><span class="r">Abelardo ►</span></div>
      </div>
      <div class="advanced-section">
        <button class="advanced-toggle" id="${b.id}-adv-btn" onclick="toggleAdv('${b.id}')">+ Abstención y voto en blanco ▾</button>
        <div class="advanced-body" id="${b.id}-adv">
          <div class="ctrl"><div class="lab"><span>Se queda en casa</span><span class="val" id="${b.id}-av">0%</span></div>
            <input type="range" id="${b.id}-abst" min="0" max="100" step="1" value="0"></div>
          <div class="ctrl"><div class="lab"><span>Vota en blanco</span><span class="val blk" id="${b.id}-bv">0%</span></div>
            <input type="range" class="blank" id="${b.id}-blank" min="0" max="60" step="1" value="0"></div>
        </div>
      </div>
    </div>`;
```

**Note on presets with hidden sliders:** `applyState()` calls `document.getElementById(b.id+"-abst").value = ...` and `document.getElementById(b.id+"-blank").value = ...`. These inputs now live inside a collapsed `.advanced-body`, but setting `.value` on a hidden input still works correctly. Presets like "Abstención alta" that set non-zero abst/blank values will update the hidden sliders silently — the numbers are correct, the user just won't see the values until they open the accordion. This is intentional behavior.

- [ ] **Step 3: Add `toggleAdv` function to the JS (after `syncBloc`)**

Add immediately after the closing brace of `syncBloc`:
```javascript
function toggleAdv(id){
  const body=document.getElementById(id+"-adv");
  const btn=document.getElementById(id+"-adv-btn");
  const open=body.classList.toggle("open");
  btn.textContent=open?"− Abstención y voto en blanco ▲":"+ Abstención y voto en blanco ▾";
}
```

---

### Task 13: Wrap retention + machine sections in a global advanced accordion

**Files:**
- Modify: `index.html` (HTML body — the two sections after `#blocs`)

The sections "Participación y bases de los finalistas" and "Maquinaria y movilización" (plus `.regions` and `.note`) should be tucked into a collapsible accordion so the page is simpler by default.

- [ ] **Step 1: Wrap lines 232–261 in advanced accordion**

Lines 232–261 cover: the "Participación y bases" section title, the retA/retC controls, the "Maquinaria" section title, the machine bloc, `.regions`, and `.note`. Line 263 starts `<footer>` — do NOT include it.

Replace lines 232–261 with this full block (existing content moved inside `.advanced-body`):
```html
<div style="margin-top:18px">
  <button class="advanced-toggle" id="globalAdv-btn" onclick="toggleGlobalAdv()" style="font-size:13px;padding:12px;border:1px solid var(--line);border-radius:12px;background:var(--panel);width:100%">
    ＋ Ajustes detallados: abstención por candidato, retención de bases, maquinaria ▾
  </button>
  <div class="advanced-body" id="globalAdv">

  <div class="sec-title">Participación y bases de los finalistas</div>
  <div class="controls">
    <div class="bloc">
      <div class="bloc-head"><div class="bloc-name">Retorno de la base de Abelardo<small>Qué tanto de sus 10.3M de primera vuelta vuelve a las urnas, o crece, el 21 de junio.</small></div></div>
      <div class="ctrl"><div class="lab"><span>Retención</span><span class="val" id="retA-v">100%</span></div>
      <input type="range" id="retA" min="80" max="115" step="1" value="100"></div>
    </div>
    <div class="bloc">
      <div class="bloc-head"><div class="bloc-name">Retorno de la base de Cepeda<small>El petrismo con la maquinaria del Estado puede crecer, o desinflarse si cunde el desánimo.</small></div></div>
      <div class="ctrl"><div class="lab"><span>Retención</span><span class="val" id="retC-v">100%</span></div>
      <input type="range" id="retC" min="80" max="115" step="1" value="100"></div>
    </div>
  </div>

  <div class="sec-title">Maquinaria y movilización</div>
  <div class="bloc">
    <div class="bloc-head"><div class="bloc-name">Factor maquinaria
      <small>Efecto neto de clientelismo y movilización transaccional, concentrado en Costa Caribe y Pacífico. En la práctica rara vez mueve más de 2 a 4 puntos a nivel nacional, pero ahí se decide un margen estrecho.</small></div></div>
    <div class="ctrl"><div class="lab"><span>Inclinación neta</span><span class="val" id="mval">0.0 pts</span></div>
      <input type="range" class="split" id="machine" min="-5" max="5" step="0.5" value="0">
      <div class="ends"><span class="l">◄ favorece a Cepeda</span><span class="r">favorece a Abelardo ►</span></div>
    </div>
  </div>

  <div class="regions">
    <div class="reg c"><h4>Fortines de Cepeda</h4><p>Pacífico (Chocó, Cauca, Nariño, Valle), Caribe de la Guajira a Sucre, y el sur profundo. Petrismo de convicción más palanca del Estado.</p></div>
    <div class="reg a"><h4>Fortines de Abelardo</h4><p>Antioquia arrasada (Medellín 55%), Eje Cafetero, Santanderes e interior andino. Más Córdoba, su tierra natal, como voto de hijo de la región.</p></div>
  </div>

  <div class="note">El choque está en la Costa: el Estado petrista empuja por Cepeda, pero De La Espriella es cordobés y reclama afinidad regional sobre el mismo territorio. Por eso esos departamentos son los más elásticos en el mapa.</div>

  </div>
</div>
```

- [ ] **Step 2: Add `toggleGlobalAdv` function to JS (after `toggleAdv`)**

```javascript
function toggleGlobalAdv(){
  const body=document.getElementById("globalAdv");
  const btn=document.getElementById("globalAdv-btn");
  const open=body.classList.toggle("open");
  btn.textContent=open?"− Ajustes detallados ▲":"＋ Ajustes detallados: abstención por candidato, retención de bases, maquinaria ▾";
}
```

---

### Task 14: Update presets, add flag footer, update meta + commit

**Files:**
- Modify: `index.html` (presets HTML + PRESETS JS object + meta tags + footer area)

- [ ] **Step 1: Rename "referendo" preset in HTML**

Old:
```html
<button data-p="referendo">Referendo anti-Tigre</button>
```
New:
```html
<button data-p="voto_castigo">Voto de castigo</button>
```

- [ ] **Step 2: Rename preset key in JS `PRESETS` object (line 284)**

Old:
```javascript
  referendo:  {paloma:[30,70,5], fajardo:[28,30,25], lopez:[18,20,18], otros:[40,45,12], retA:99,  retC:108, machine:-3},
```
New:
```javascript
  voto_castigo:{paloma:[30,70,5], fajardo:[28,30,25], lopez:[18,20,18], otros:[40,45,12], retA:99,  retC:108, machine:-3},
```

- [ ] **Step 3: Add flag footer before closing `.wrap` div**

Before `<div class="toast" id="toast">`, add:
```html
<div class="flag-footer"></div>
```

- [ ] **Step 4: Update social meta description (line 9)**

Old:
```html
<meta name="description" content="Modelo interactivo de segunda vuelta presidencial Colombia 2026. De La Espriella 43.7% vs Cepeda 40.9% · mueve los supuestos y ve a dónde van los votos.">
```
New:
```html
<meta name="description" content="Modelo interactivo de segunda vuelta presidencial Colombia 2026. De La Espriella 43.7% vs Cepeda 40.9% · mueve los supuestos y ve a dónde van los votos libres.">
```

- [ ] **Step 5: Search for remaining "balotaje" occurrences and replace**

Run in terminal:
```bash
grep -n "alotaje\|huérfanos\|huerfanos" index.html
```
Replace any found occurrences: "balotaje" → "segunda vuelta", "votos huérfanos" → "votos libres".

- [ ] **Step 6: Verify preset works**

Open browser. Click "Voto de castigo". Numbers should update. Click "Caso base". Should reset.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: candidate hero, how-to banner, simplified sliders, Colombia flag stripes, Colombian terminology"
```

---

## Chunk 4: JavaScript — localStorage & Final Wiring

---

### Task 15: Add `localStorage` dismiss for how-to banner

**Files:**
- Modify: `index.html` (the `init` IIFE, lines 479–484)

- [ ] **Step 1: Replace the `init` IIFE**

Old:
```javascript
(function init(){
  const h=location.hash;
  if(h.startsWith("#s=")){const cfg=decodeState(h.slice(3));
    if(cfg){document.querySelectorAll("#presets button").forEach(b=>b.classList.remove("on"));applyState(cfg);return;}}
  applyState(PRESETS.base);
})();
```

New:
```javascript
(function init(){
  // How-to banner: hide if previously dismissed
  if(localStorage.getItem("howToDismissed")==="1"){
    const ht=document.getElementById("howTo");
    if(ht) ht.style.display="none";
  }
  document.getElementById("howToDismiss").addEventListener("click",()=>{
    localStorage.setItem("howToDismissed","1");
    document.getElementById("howTo").style.display="none";
  });
  // URL state restore
  const h=location.hash;
  if(h.startsWith("#s=")){const cfg=decodeState(h.slice(3));
    if(cfg){document.querySelectorAll("#presets button").forEach(b=>b.classList.remove("on"));applyState(cfg);return;}}
  applyState(PRESETS.base);
})();
```

**Note:** `document.getElementById("howToDismiss")` is called unconditionally. The element is always present in the DOM (it's a child of `#howTo`, not removed, only hidden). If the banner HTML is ever removed, this will throw. Both `#howTo` and `#howToDismiss` must remain in the HTML as long as this JS exists.

- [ ] **Step 2: Verify persistence**

1. Open browser → banner is visible.
2. Click "Entendido, no volver a mostrar ✕" → banner disappears.
3. Reload page → banner stays hidden.
4. Open DevTools → Application → Local Storage → confirm key `howToDismissed = 1`.
5. Delete the key in DevTools → reload → banner reappears.

---

### Task 16: Final verification checklist + commit

**Files:**
- Modify: `index.html` (any remaining fixes found during verification)

- [ ] **Step 1: Full visual + functional checklist**

Open `index.html` in browser (use a local server or open directly). Verify each item:

| Check | Expected |
|---|---|
| Flag stripe | Visible at top: yellow left half, blue 25%, red 25% |
| Background | Light cream (#F5F3EF), white cards |
| Abelardo photo | Loads, circular, blue border |
| Cepeda photo | Loads, circular, red border |
| How-to banner | Visible on fresh load (no localStorage key) |
| Banner dismiss | Click ✕ → hides, stays hidden after reload |
| Caso base preset | Active (blue) on load, pa≈52.4%, pc≈46.7% |
| Bar | Doesn't quite fill 100% — tiny gap right = blank votes |
| "50%" label | At ~1/3 from left of histogram, not at center |
| Voto de castigo | Preset button works, numbers update |
| Per-bloc split slider | Only split visible by default |
| "+ Abstención" toggle | Opens/closes abstention + blank sliders |
| Global advanced toggle | Opens/closes retención + maquinaria sections |
| Sharing URL | Encodes/decodes state, URL updates in address bar |
| Mobile (560px) | Candidate cards stack, VS badge hidden |
| Console | Zero errors |

- [ ] **Step 2: Fix any issues found**

Address any failing checks before committing.

- [ ] **Step 3: Final commit**

```bash
git add index.html
git commit -m "feat: localStorage banner dismiss, final wiring and verification pass"
```
