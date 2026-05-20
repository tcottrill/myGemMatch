# Bejeweled Clone Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a self-contained `index.html` Bejeweled-style match-3 game (8×8, 7 colors, 3 power gems, cascades, endless mode) with WebAudio, hint system, best-score persistence, and full PWA install support, reusing the gem SVGs from the existing `bejeweled.html`.

**Architecture:** DOM+canvas hybrid. CSS chrome for header/menu/dialogs; a single canvas for the board, rendered at DPR ≤ 2. Gems are rasterized once at boot from extracted SVG symbols, then `drawImage`-blitted each frame. All animations driven by one self-stopping `requestAnimationFrame` loop. State machine: `LOADING → MENU → PLAY (IDLE/SWAP/RESOLVE) → MENU`.

**Tech Stack:** Plain HTML/CSS/ES2020 in one file. Canvas 2D. Pointer Events. WebAudio (synth only). `DecompressionStream` not needed (no large text blobs). `localStorage` for persistence. PWA: `site.webmanifest`, apple-touch-icons, 192/512 maskable icons generated via ImageMagick.

**Verification model:** This is a browser game; there is no automated test runner. Each task's verification step is "open `index.html`, perform the listed user actions, observe the listed result." Use Claude Preview MCP (`mcp__Claude_Preview__preview_start`) to load the file and `preview_screenshot` / `preview_console_logs` to verify.

---

## File Structure

```
myBejeweled/
├── index.html                  # the entire game (created in Task 1, grown through Tasks 2–9)
├── bejeweled.html              # original gem source, never modified
├── icon-master.svg             # vector source for all icons (Task 10)
├── icon-192.png                # generated
├── icon-512.png                # generated
├── apple-touch-icon.png        # 180×180, generated
├── apple-touch-icon-167x167.png
├── apple-touch-icon-152x152.png
├── favicon.ico
├── favicon-16x16.png
├── favicon-32x32.png
├── site.webmanifest            # PWA manifest (Task 10)
└── README.md                   # Task 11
```

The entire game lives in `index.html`. The file is organized internally with these top-to-bottom sections (use comment banners):

1. `<head>` — PWA meta, manifest link, icons, `<style>`
2. `<body>` — DOM chrome (header, board container, menu overlay, dialogs)
3. Hidden SVG sprite (gem symbol defs)
4. `<script>` — in this order:
   - Constants (palette, board size, animation timings, scoring)
   - State (board, score, animations, etc.)
   - Persistence helpers (load/save best score, mute)
   - Audio module (lazy WebAudio + sound recipes)
   - Sprite baking (SVG → Image)
   - Canvas setup (DPR clamp, resize debounce)
   - Animation loop (`ensureAnimLoop`, easing)
   - Render (`draw`, gem render, overlays)
   - Game logic (init board, match detect, gravity, deadlock, hint)
   - Input handling (Pointer Events, keyboard)
   - State machine (menu/play transitions, dialogs)
   - Boot

---

## Task 1: HTML skeleton + PWA shell + sprite extraction

**Files:**
- Create: `index.html`

- [ ] **Step 1: Read the source gem SVGs**

Read `bejeweled.html` lines 1–122 to capture all gem `<symbol>` definitions and the `<linearGradient id="hyper_gradient">`, `<radialGradient id="fire_gradient">`, `<linearGradient id="star_gradient">` defs. We need: `tile_red`, `tile_orange`, `tile_yellow`, `tile_green`, `tile_blue`, `tile_purple`, `tile_white`, `tile_hyper` (we'll synthesize fire/star overlays in canvas rather than embed the animated SVGs).

- [ ] **Step 2: Create `index.html` skeleton**

Per the skill's `references/pwa-setup.md`:

```html
<!doctype html>
<html lang="en" translate="no">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
  <meta name="google" content="notranslate">
  <meta name="apple-mobile-web-app-capable" content="yes">
  <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
  <meta name="apple-mobile-web-app-title" content="Bejeweled">
  <meta name="theme-color" content="#1a1230">
  <title>Bejeweled</title>
  <link rel="manifest" href="site.webmanifest">
  <link rel="apple-touch-icon" sizes="180x180" href="apple-touch-icon.png">
  <link rel="apple-touch-icon" sizes="167x167" href="apple-touch-icon-167x167.png">
  <link rel="apple-touch-icon" sizes="152x152" href="apple-touch-icon-152x152.png">
  <link rel="icon" type="image/png" sizes="32x32" href="favicon-32x32.png">
  <link rel="icon" type="image/png" sizes="16x16" href="favicon-16x16.png">
  <link rel="shortcut icon" href="favicon.ico">
  <style>/* fill in Step 3 */</style>
</head>
<body>
  <div id="app">
    <header id="topbar">
      <button id="menuBtn" aria-label="Menu">☰</button>
      <div class="score-block"><div class="label">SCORE</div><div id="score">0</div></div>
      <div class="score-block"><div class="label">BEST</div><div id="best">0</div></div>
      <button id="muteBtn" aria-label="Mute">🔊</button>
    </header>
    <div id="boardWrap">
      <canvas id="board" width="640" height="640"></canvas>
    </div>
  </div>
  <div id="menuOverlay" class="overlay">
    <div class="panel">
      <h1>Bejeweled</h1>
      <button id="newGameBtn">New Game</button>
    </div>
  </div>
  <div id="dialogOverlay" class="overlay hidden">
    <div class="panel" id="dialogPanel"></div>
  </div>
  <svg id="gem-sprite" style="position:absolute;width:0;height:0;overflow:hidden" aria-hidden="true">
    <!-- defs + 7 color symbols + tile_hyper symbol pasted from bejeweled.html, Step 4 -->
  </svg>
  <script>/* fill in Step 5+ */</script>
</body>
</html>
```

- [ ] **Step 3: Add CSS (dark theme, mobile-safe)**

Inside `<style>`:

```css
:root { --bg:#1a1230; --bg2:#251947; --fg:#fff; --accent:#ffd84a; }
* { box-sizing:border-box; }
html, body { margin:0; height:100%; background:var(--bg); color:var(--fg);
  font-family: system-ui, -apple-system, Segoe UI, Roboto, sans-serif;
  overscroll-behavior:none; -webkit-tap-highlight-color:transparent; user-select:none; }
#app { display:flex; flex-direction:column; height:100vh; height:100dvh;
  padding-top:env(safe-area-inset-top); padding-bottom:env(safe-area-inset-bottom); }
#topbar { display:flex; align-items:center; gap:12px; padding:10px 14px;
  background:var(--bg2); border-bottom:1px solid rgba(255,255,255,0.08); }
#topbar button { background:transparent; color:var(--fg); border:1px solid rgba(255,255,255,0.15);
  border-radius:8px; padding:8px 12px; font-size:18px; cursor:pointer; }
#topbar button:active { transform:scale(0.96); }
.score-block { flex:1; text-align:center; }
.score-block .label { font-size:11px; opacity:0.6; letter-spacing:1px; }
.score-block #score, .score-block #best { font-size:20px; font-weight:600; color:var(--accent); }
#boardWrap { flex:1; display:flex; align-items:center; justify-content:center; padding:12px;
  min-height:0; }
#board { touch-action:none; max-width:100%; max-height:100%;
  background:#0e0820; border-radius:12px;
  box-shadow:0 8px 30px rgba(0,0,0,0.5), inset 0 0 0 1px rgba(255,255,255,0.06); }
.overlay { position:fixed; inset:0; background:rgba(0,0,0,0.7); display:flex;
  align-items:center; justify-content:center; z-index:10; backdrop-filter:blur(4px); }
.overlay.hidden { display:none; }
.panel { background:var(--bg2); padding:28px 32px; border-radius:14px; min-width:260px;
  text-align:center; box-shadow:0 10px 40px rgba(0,0,0,0.6); }
.panel h1 { margin:0 0 16px; font-size:28px; letter-spacing:2px; color:var(--accent); }
.panel button { display:block; width:100%; margin:8px 0; padding:12px;
  background:var(--accent); color:#1a1230; border:none; border-radius:8px;
  font-size:16px; font-weight:600; cursor:pointer; }
.panel button.secondary { background:transparent; color:var(--fg);
  border:1px solid rgba(255,255,255,0.2); }
#toast { position:fixed; bottom:18%; left:50%; transform:translateX(-50%);
  background:rgba(0,0,0,0.8); color:var(--fg); padding:10px 16px; border-radius:8px;
  pointer-events:none; opacity:0; transition:opacity 0.2s; z-index:20; }
#toast.show { opacity:1; }
```

Add `<div id="toast"></div>` inside `<body>`.

- [ ] **Step 4: Paste gem sprite into `<svg id="gem-sprite">`**

Copy from `bejeweled.html` (read first, paste verbatim):
- The three `<defs>` blocks containing `hyper_gradient`, `fire_gradient`, `star_gradient` (still useful for the `tile_hyper` symbol, which uses `url(#hyper_gradient)`).
- The seven color `<symbol>` blocks: `tile_red`, `tile_orange`, `tile_yellow`, `tile_green`, `tile_blue`, `tile_purple`, `tile_white`.
- The `<symbol id="tile_hyper">` block.

Wrap them in `<defs>` inside `#gem-sprite`. Note: source IDs don't collide with each other, but namespace them by prefixing with `g_` (e.g., `g_tile_red`, `g_hyper_gradient`) and update all internal `url(#hyper_gradient)` references inside the pasted symbols to `url(#g_hyper_gradient)`. This is the skill's namespace-safety pass.

- [ ] **Step 5: Add minimal boot script**

```html
<script>
'use strict';
const GEMS = ['red','orange','yellow','green','blue','purple','white'];
const N = 8;  // board dimension
window.addEventListener('DOMContentLoaded', () => {
  console.log('Bejeweled boot: gems', GEMS, 'board', N);
});
</script>
```

- [ ] **Step 6: Verify in browser**

Use Claude Preview:
1. `preview_start` with the `index.html` path.
2. `preview_screenshot` — expect dark page with title "Bejeweled", menu overlay visible, "New Game" button visible.
3. `preview_console_logs` — expect the boot log line.
4. Inspect the DOM: confirm `#gem-sprite` is in the document (`preview_eval` with `document.querySelectorAll('#gem-sprite symbol').length` → expect `8`).

- [ ] **Step 7: Stop here. No commits.** This project is not a git repo. Skip the commit step for every task in this plan.

---

## Task 2: Rasterize gem sprites + render a static board

**Files:**
- Modify: `index.html` (script block)

- [ ] **Step 1: Add sprite baking function**

Each symbol in `#gem-sprite` has a 50×50 viewBox. To turn one into a drawable `Image`, wrap its inner SVG as a standalone `<svg viewBox="-25 -25 50 50">` document, serialize, and create an Image from a `data:image/svg+xml;utf8,...` URL. The wrapper must include the gradient defs the symbol's `<use>` references.

Add to the script:

```js
const SPRITE_PX = 128;   // rasterize at this resolution, scale down per cell
const gemImages = {};    // { red: HTMLImageElement, ... , hyper: HTMLImageElement }

function bakeSprites() {
  const sprite = document.getElementById('gem-sprite');
  const defsHTML = sprite.querySelector('defs').innerHTML;
  const promises = [];
  for (const name of [...GEMS, 'hyper']) {
    const sym = document.getElementById('g_tile_' + name);
    if (!sym) { console.warn('missing symbol', name); continue; }
    const innerHTML = sym.innerHTML;
    const svgStr = `<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink"
      viewBox="-25 -25 50 50" width="${SPRITE_PX}" height="${SPRITE_PX}">
      <defs>${defsHTML}</defs>${innerHTML}</svg>`;
    const url = 'data:image/svg+xml;utf8,' + encodeURIComponent(svgStr);
    const img = new Image();
    const p = new Promise((res, rej) => {
      img.onload = () => res();
      img.onerror = () => rej(new Error('svg load failed: ' + name));
    });
    img.src = url;
    gemImages[name] = img;
    promises.push(p);
  }
  return Promise.all(promises);
}
```

- [ ] **Step 2: Add canvas setup with DPR clamp + debounced resize**

```js
const MAX_RENDER_DPR = 2;
const canvas = document.getElementById('board');
const ctx = canvas.getContext('2d');
let cssSize = 0, cellPx = 0, dpr = 1;

function setupCanvas() {
  const wrap = document.getElementById('boardWrap');
  const w = wrap.clientWidth, h = wrap.clientHeight;
  cssSize = Math.floor(Math.min(w, h));
  dpr = Math.min(window.devicePixelRatio || 1, MAX_RENDER_DPR);
  canvas.style.width = cssSize + 'px';
  canvas.style.height = cssSize + 'px';
  canvas.width = Math.floor(cssSize * dpr);
  canvas.height = Math.floor(cssSize * dpr);
  cellPx = cssSize / N;
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
  draw();
}

let resizeTimer = null;
function scheduleResize(delay = 120) {
  clearTimeout(resizeTimer);
  resizeTimer = setTimeout(setupCanvas, delay);
}
window.addEventListener('resize', () => scheduleResize(120));
window.addEventListener('orientationchange', () => scheduleResize(260));
```

- [ ] **Step 3: Add board init + simple draw**

```js
let board = [];  // board[r][c] = { color: 0..6, special: null|'fire'|'star'|'hyper' }

function makeGem(color, special = null) { return { color, special }; }

function newBoard() {
  board = [];
  for (let r = 0; r < N; r++) {
    const row = [];
    for (let c = 0; c < N; c++) {
      // pick a color that doesn't immediately complete a 3-in-a-row
      let tries = 0, color;
      do {
        color = Math.floor(Math.random() * GEMS.length);
        tries++;
      } while (tries < 20 && completesRunAt(row, board, r, c, color));
      row.push(makeGem(color));
    }
    board.push(row);
  }
}

function completesRunAt(row, prevRows, r, c, color) {
  // left two cells same color
  if (c >= 2 && row[c-1].color === color && row[c-2].color === color) return true;
  // top two cells same color
  if (r >= 2 && prevRows[r-1][c].color === color && prevRows[r-2][c].color === color) return true;
  return false;
}

function draw() {
  if (!cssSize) return;
  ctx.clearRect(0, 0, cssSize, cssSize);
  // checkerboard background
  for (let r = 0; r < N; r++) {
    for (let c = 0; c < N; c++) {
      ctx.fillStyle = ((r + c) & 1) ? '#1f1638' : '#251947';
      ctx.fillRect(c * cellPx, r * cellPx, cellPx, cellPx);
    }
  }
  // gems
  for (let r = 0; r < N; r++) {
    for (let c = 0; c < N; c++) {
      const g = board[r][c];
      if (!g) continue;
      const img = gemImages[g.special === 'hyper' ? 'hyper' : GEMS[g.color]];
      if (img && img.complete && img.naturalWidth > 0) {
        ctx.drawImage(img, c * cellPx, r * cellPx, cellPx, cellPx);
      }
    }
  }
}
```

- [ ] **Step 4: Wire boot**

Replace the boot listener with:

```js
window.addEventListener('DOMContentLoaded', async () => {
  try { await bakeSprites(); } catch (e) { console.error(e); }
  newBoard();
  setupCanvas();
});
```

- [ ] **Step 5: Verify**

`preview_start`, dismiss the menu by injecting `document.getElementById('menuOverlay').classList.add('hidden')` via `preview_eval`, then `preview_screenshot`. Expect: 8×8 grid of colored gems, no immediate 3-in-a-row visible. Run `preview_console_logs` — expect no errors.

If a gem renders blank, check `gemImages[name].naturalWidth > 0` per the skill's pitfalls — likely a malformed sprite svg (gradient ID still missing prefix).

---

## Task 3: Pointer input + swap animation (no match logic yet)

**Files:**
- Modify: `index.html` (script block)

- [ ] **Step 1: Add animation loop scaffolding**

```js
const anims = [];   // each: { kind, t0, dur, ...state }
let rafHandle = null;

function easeOutCubic(t) { return 1 - Math.pow(1 - t, 3); }

function ensureAnimLoop() {
  if (rafHandle !== null) return;
  const tick = (now) => {
    let active = false;
    for (let i = anims.length - 1; i >= 0; i--) {
      const a = anims[i];
      const t = (now - a.t0) / a.dur;
      if (t >= 1) {
        if (a.onDone) a.onDone();
        anims.splice(i, 1);
      } else {
        active = true;
      }
    }
    draw();
    if (active || anims.length) {
      rafHandle = requestAnimationFrame(tick);
    } else {
      rafHandle = null;
      onAnimsSettled();
    }
  };
  rafHandle = requestAnimationFrame(tick);
}

function onAnimsSettled() { /* filled in later tasks */ }
```

- [ ] **Step 2: Add swap animation kind + draw integration**

Add a swap animation:

```js
function animateSwap(r1, c1, r2, c2, onDone) {
  const id = Math.random();
  anims.push({
    kind: 'swap', t0: performance.now(), dur: 200,
    r1, c1, r2, c2, onDone, id,
  });
  ensureAnimLoop();
}
```

Modify `draw()`: before drawing each gem, check if any active `swap` animation references its cell. If so, offset its position by interpolating toward the partner cell.

```js
function gemRenderOffset(r, c, now) {
  for (const a of anims) {
    if (a.kind !== 'swap') continue;
    const t = Math.min(1, (now - a.t0) / a.dur);
    const e = easeOutCubic(t);
    if (a.r1 === r && a.c1 === c) {
      return { dx: (a.c2 - a.c1) * cellPx * e, dy: (a.r2 - a.r1) * cellPx * e };
    }
    if (a.r2 === r && a.c2 === c) {
      return { dx: (a.c1 - a.c2) * cellPx * e, dy: (a.r1 - a.r2) * cellPx * e };
    }
  }
  return { dx: 0, dy: 0 };
}
```

Update the gem-drawing loop in `draw()`:

```js
const now = performance.now();
for (let r = 0; r < N; r++) {
  for (let c = 0; c < N; c++) {
    const g = board[r][c];
    if (!g) continue;
    const off = gemRenderOffset(r, c, now);
    const img = gemImages[g.special === 'hyper' ? 'hyper' : GEMS[g.color]];
    if (img && img.complete && img.naturalWidth > 0) {
      ctx.drawImage(img, c * cellPx + off.dx, r * cellPx + off.dy, cellPx, cellPx);
    }
  }
}
```

- [ ] **Step 3: Add selection highlight to draw**

```js
let selected = null;  // {r, c}

// in draw(), after gems:
if (selected) {
  ctx.strokeStyle = '#ffd84a';
  ctx.lineWidth = 3;
  ctx.strokeRect(selected.c * cellPx + 2, selected.r * cellPx + 2,
                 cellPx - 4, cellPx - 4);
}
```

- [ ] **Step 4: Add Pointer Events**

```js
let inputLocked = false;        // true while RESOLVE / animating

function cellFromEvent(ev) {
  const rect = canvas.getBoundingClientRect();
  const x = ev.clientX - rect.left, y = ev.clientY - rect.top;
  if (x < 0 || y < 0 || x >= cssSize || y >= cssSize) return null;
  return { r: Math.floor(y / cellPx), c: Math.floor(x / cellPx) };
}

function isAdjacent(a, b) {
  return (Math.abs(a.r - b.r) + Math.abs(a.c - b.c)) === 1;
}

function swapCells(r1, c1, r2, c2) {
  const tmp = board[r1][c1];
  board[r1][c1] = board[r2][c2];
  board[r2][c2] = tmp;
}

function attemptSwap(a, b) {
  if (inputLocked) return;
  inputLocked = true;
  selected = null;
  swapCells(a.r, a.c, b.r, b.c);
  // until Task 4, just animate visually then revert — placeholder
  animateSwap(a.r, a.c, b.r, b.c, () => {
    // Task 4 will validate and either keep or revert; for now keep.
    inputLocked = false;
    draw();
  });
}

let pointerDownCell = null;
canvas.addEventListener('pointerdown', (ev) => {
  ev.preventDefault();
  canvas.setPointerCapture(ev.pointerId);
  if (inputLocked) return;
  const cell = cellFromEvent(ev);
  if (!cell) return;
  pointerDownCell = cell;
  if (selected && isAdjacent(selected, cell)) {
    attemptSwap(selected, cell);
  } else {
    selected = cell;
    draw();
  }
});

canvas.addEventListener('pointerup', (ev) => {
  if (!pointerDownCell || inputLocked) { pointerDownCell = null; return; }
  const end = cellFromEvent(ev);
  if (end && !sameCell(end, pointerDownCell) && isAdjacent(pointerDownCell, end)) {
    // drag-swipe swap
    selected = null;
    attemptSwap(pointerDownCell, end);
  }
  pointerDownCell = null;
});

canvas.addEventListener('pointercancel', () => {
  pointerDownCell = null;
});

function sameCell(a, b) { return a.r === b.r && a.c === b.c; }
```

- [ ] **Step 5: Hide menu on New Game click**

```js
document.getElementById('newGameBtn').addEventListener('click', () => {
  document.getElementById('menuOverlay').classList.add('hidden');
  newBoard();
  draw();
});
```

- [ ] **Step 6: Verify**

`preview_start`. Click New Game. Click one gem → expect yellow outline. Click an adjacent gem → expect a 200ms swap animation; the two gems exchange positions (and stay swapped, since Task 4 isn't done yet). Click a non-adjacent gem after selecting → expect selection moves to the new gem. Check `preview_console_logs` for errors.

---

## Task 4: Match detection + cascade resolution

**Files:**
- Modify: `index.html` (script block)

- [ ] **Step 1: Add match-finding**

```js
// Returns array of "match groups": each is an array of {r,c} cells that should clear.
// A group is a run of 3+ same-color in a row or column (no specials considered here).
function findMatches() {
  const groups = [];
  // rows
  for (let r = 0; r < N; r++) {
    let runStart = 0;
    for (let c = 1; c <= N; c++) {
      const sameAsPrev = c < N
        && board[r][c] && board[r][runStart]
        && board[r][c].color === board[r][runStart].color;
      if (!sameAsPrev) {
        const len = c - runStart;
        if (len >= 3) {
          const g = [];
          for (let cc = runStart; cc < c; cc++) g.push({ r, c: cc });
          groups.push({ kind: 'row', len, cells: g });
        }
        runStart = c;
      }
    }
  }
  // columns
  for (let c = 0; c < N; c++) {
    let runStart = 0;
    for (let r = 1; r <= N; r++) {
      const sameAsPrev = r < N
        && board[r][c] && board[runStart][c]
        && board[r][c].color === board[runStart][c].color;
      if (!sameAsPrev) {
        const len = r - runStart;
        if (len >= 3) {
          const g = [];
          for (let rr = runStart; rr < r; rr++) g.push({ r: rr, c });
          groups.push({ kind: 'col', len, cells: g });
        }
        runStart = r;
      }
    }
  }
  return groups;
}
```

- [ ] **Step 2: Add gravity + refill**

```js
function applyGravityAndRefill() {
  for (let c = 0; c < N; c++) {
    // compact non-null gems downward
    let writeRow = N - 1;
    for (let r = N - 1; r >= 0; r--) {
      if (board[r][c]) {
        if (writeRow !== r) {
          board[writeRow][c] = board[r][c];
          board[r][c] = null;
        }
        writeRow--;
      }
    }
    // fill the rest with new random gems
    for (let r = writeRow; r >= 0; r--) {
      board[r][c] = makeGem(Math.floor(Math.random() * GEMS.length));
    }
  }
}
```

(Task 6 will add a "falling" animation for these; for Task 4, gems pop into place.)

- [ ] **Step 3: Add clear animation kind**

```js
function animateClear(cells, onDone) {
  // mark each gem as "clearing" so draw shrinks/fades it
  const ids = cells.map(c => ({ r: c.r, c: c.c }));
  anims.push({
    kind: 'clear', t0: performance.now(), dur: 250, cells: ids, onDone,
  });
  ensureAnimLoop();
}
```

In `draw()`, replace the gem render with awareness of clear anims:

```js
function clearProgressAt(r, c, now) {
  for (const a of anims) {
    if (a.kind !== 'clear') continue;
    for (const cell of a.cells) {
      if (cell.r === r && cell.c === c) {
        return Math.min(1, (now - a.t0) / a.dur);
      }
    }
  }
  return 0;
}

// in draw() gem loop:
const p = clearProgressAt(r, c, now);
const scale = 1 - p;
const alpha = 1 - p;
if (alpha <= 0) continue;
ctx.save();
ctx.globalAlpha = alpha;
const cx = c * cellPx + cellPx / 2 + off.dx;
const cy = r * cellPx + cellPx / 2 + off.dy;
const size = cellPx * scale;
ctx.drawImage(img, cx - size / 2, cy - size / 2, size, size);
ctx.restore();
```

- [ ] **Step 4: Replace `attemptSwap` with full resolve loop**

```js
let score = 0;
let cascadeLevel = 0;

function updateScoreUI() { document.getElementById('score').textContent = score; }

function attemptSwap(a, b) {
  if (inputLocked) return;
  inputLocked = true;
  selected = null;
  swapCells(a.r, a.c, b.r, b.c);
  const matches = findMatches();
  if (matches.length === 0) {
    // illegal — animate back
    animateSwap(a.r, a.c, b.r, b.c, () => {
      swapCells(a.r, a.c, b.r, b.c);  // revert data
      animateSwap(a.r, a.c, b.r, b.c, () => {
        inputLocked = false;
      });
    });
    return;
  }
  // legal — animate forward then resolve cascades
  // We already swapped data; play swap anim from old→new visually.
  swapCells(a.r, a.c, b.r, b.c);  // temporarily unswap data so anim moves the original gems
  animateSwap(a.r, a.c, b.r, b.c, () => {
    swapCells(a.r, a.c, b.r, b.c);  // re-apply swap
    cascadeLevel = 0;
    resolveCascades();
  });
}

function resolveCascades() {
  const matches = findMatches();
  if (matches.length === 0) {
    inputLocked = false;
    onAnimsSettled();
    return;
  }
  cascadeLevel++;
  // collect all clearing cells (dedupe)
  const clearing = new Map();
  let scoreThisStep = 0;
  for (const m of matches) {
    const base = m.len === 3 ? 30 : m.len === 4 ? 60 : 100;
    scoreThisStep += base;
    for (const cell of m.cells) clearing.set(cell.r + ',' + cell.c, cell);
  }
  const mul = Math.pow(1.5, cascadeLevel - 1);
  score += Math.floor(scoreThisStep * mul);
  updateScoreUI();
  const cells = [...clearing.values()];
  animateClear(cells, () => {
    for (const cell of cells) board[cell.r][cell.c] = null;
    applyGravityAndRefill();
    setTimeout(resolveCascades, 80);  // brief beat between cascades
  });
}
```

(`onAnimsSettled` already a no-op; Task 7 hooks deadlock detection here.)

- [ ] **Step 4a: Update revert animation correctly**

The revert in Step 4 has a subtle bug — re-read it. The animation should: swap data, animate forward, on done animate back, on done swap data back. Replace the `matches.length === 0` branch with:

```js
if (matches.length === 0) {
  // We swapped data; animate forward, then back, then unswap.
  swapCells(a.r, a.c, b.r, b.c);  // unswap data; visual will move it
  animateSwap(a.r, a.c, b.r, b.c, () => {
    // pause briefly, then animate back
    swapCells(a.r, a.c, b.r, b.c);  // pretend to commit
    animateSwap(a.r, a.c, b.r, b.c, () => {
      swapCells(a.r, a.c, b.r, b.c);  // unswap again to final original state
      inputLocked = false;
      draw();
    });
  });
  return;
}
```

(Verify by trace: start AB → swap data → BA → no match → unswap → AB → anim swap (visually BA) → swap data → BA → anim swap (visually AB) → unswap → AB. Final state matches starting state. ✓)

- [ ] **Step 5: Verify**

Open the page. Click New Game. Swap two adjacent gems that form a match: expect them to swap, the matched run to fade-scale out, the column to refill from above (instantly, no fall anim yet), and the score to increase. Try an illegal swap (no match): expect the two gems to swap and immediately swap back. Test a cascade: arrange the board (or be lucky) so a refill creates another match; expect the score multiplier to grow per chain (check console).

Add temporary `console.log('cascade', cascadeLevel, 'score', score)` if needed for verification, then remove.

---

## Task 5: Power gem creation + activation

**Files:**
- Modify: `index.html` (script block)

- [ ] **Step 1: Detect special-spawn during a committing swap**

Rewrite `resolveCascades` so the first cascade (level 1) computes power-gem spawns from the swap-origin cell. We need to pass the swap cells in:

```js
let lastSwap = null;  // { a:{r,c}, b:{r,c} } for the most recent committing swap

function attemptSwap(a, b) {
  if (inputLocked) return;
  inputLocked = true;
  selected = null;
  swapCells(a.r, a.c, b.r, b.c);
  // hyper activation: swapping a hyper with any colored gem clears all of that color
  const swappedHyperA = board[a.r][a.c].special === 'hyper';
  const swappedHyperB = board[b.r][b.c].special === 'hyper';
  if (swappedHyperA || swappedHyperB) {
    const hyperCell = swappedHyperA ? a : b;
    const targetCell = swappedHyperA ? b : a;
    const targetColor = board[targetCell.r][targetCell.c].color;
    // animate the swap, then clear all matching color + the hyper
    swapCells(a.r, a.c, b.r, b.c);   // unswap to play the anim
    animateSwap(a.r, a.c, b.r, b.c, () => {
      swapCells(a.r, a.c, b.r, b.c);
      const cells = [];
      for (let r = 0; r < N; r++) for (let c = 0; c < N; c++) {
        const g = board[r][c];
        if (!g) continue;
        if ((r === hyperCell.r && c === hyperCell.c) || g.color === targetColor) {
          cells.push({ r, c });
        }
      }
      lastSwap = { a, b };
      cascadeLevel = 0;
      clearAndCascade(cells);
    });
    return;
  }
  const matches = findMatches();
  if (matches.length === 0) {
    // (revert branch from Task 4)
    swapCells(a.r, a.c, b.r, b.c);
    animateSwap(a.r, a.c, b.r, b.c, () => {
      swapCells(a.r, a.c, b.r, b.c);
      animateSwap(a.r, a.c, b.r, b.c, () => {
        swapCells(a.r, a.c, b.r, b.c);
        inputLocked = false;
        draw();
      });
    });
    return;
  }
  swapCells(a.r, a.c, b.r, b.c);
  animateSwap(a.r, a.c, b.r, b.c, () => {
    swapCells(a.r, a.c, b.r, b.c);
    lastSwap = { a, b };
    cascadeLevel = 0;
    resolveCascades();
  });
}
```

- [ ] **Step 2: Refactor `resolveCascades` to optionally inject a starting clear set**

Add `clearAndCascade(cells)` used by hyper activation:

```js
function clearAndCascade(cells) {
  if (cells.length === 0) { inputLocked = false; return; }
  cascadeLevel++;
  const base = Math.max(30, cells.length * 20);
  const mul = Math.pow(1.5, cascadeLevel - 1);
  score += Math.floor(base * mul);
  updateScoreUI();
  animateClear(cells, () => {
    // before nulling, also detonate any specials in the cleared set
    detonateSpecialsAndContinue(cells);
  });
}

function detonateSpecialsAndContinue(cells) {
  // any specials present in `cells` add more cells to clear
  const extras = [];
  const seen = new Set(cells.map(c => c.r + ',' + c.c));
  for (const cell of cells) {
    const g = board[cell.r][cell.c];
    if (!g || !g.special) continue;
    if (g.special === 'fire') {
      for (let dr = -1; dr <= 1; dr++) for (let dc = -1; dc <= 1; dc++) {
        const r = cell.r + dr, c = cell.c + dc;
        if (r < 0 || r >= N || c < 0 || c >= N) continue;
        const key = r + ',' + c;
        if (!seen.has(key) && board[r][c]) { seen.add(key); extras.push({ r, c }); }
      }
    } else if (g.special === 'star') {
      for (let i = 0; i < N; i++) {
        for (const p of [{r: cell.r, c: i}, {r: i, c: cell.c}]) {
          const key = p.r + ',' + p.c;
          if (!seen.has(key) && board[p.r][p.c]) { seen.add(key); extras.push(p); }
        }
      }
    }
    // hyper detonation handled in attemptSwap (only when swapped directly)
  }
  for (const cell of cells) board[cell.r][cell.c] = null;
  if (extras.length) {
    animateClear(extras, () => {
      // recursive: extras may include more specials
      detonateSpecialsAndContinue(extras);
    });
  } else {
    applyGravityAndRefill();
    setTimeout(resolveCascades, 80);
  }
}
```

- [ ] **Step 3: Update `resolveCascades` to spawn power gems from matches**

```js
function resolveCascades() {
  const matches = findMatches();
  if (matches.length === 0) {
    inputLocked = false;
    onAnimsSettled();
    return;
  }
  cascadeLevel++;
  const clearing = new Map();          // key → {r,c}
  const cellGroupMap = new Map();      // key → groups it belongs to
  let scoreThisStep = 0;
  for (const m of matches) {
    const base = m.len === 3 ? 30 : m.len === 4 ? 60 : 100;
    scoreThisStep += base;
    for (const cell of m.cells) {
      const k = cell.r + ',' + cell.c;
      clearing.set(k, cell);
      if (!cellGroupMap.has(k)) cellGroupMap.set(k, []);
      cellGroupMap.get(k).push(m);
    }
  }
  // power-gem spawn: only at level 1 (the player's swap), at the swap-origin cell
  const spawns = [];   // [{cell, special, color}]
  if (cascadeLevel === 1 && lastSwap) {
    for (const origin of [lastSwap.a, lastSwap.b]) {
      const k = origin.r + ',' + origin.c;
      const groups = cellGroupMap.get(k);
      if (!groups) continue;
      const color = board[origin.r][origin.c].color;
      // T/L: two groups containing this cell → hyper
      if (groups.length >= 2) {
        spawns.push({ cell: origin, special: 'hyper', color });
      } else if (groups[0].len >= 5) {
        spawns.push({ cell: origin, special: 'star', color });
      } else if (groups[0].len === 4) {
        spawns.push({ cell: origin, special: 'fire', color });
      }
    }
  }
  // remove spawn cells from clearing so they survive as the new special
  for (const s of spawns) clearing.delete(s.cell.r + ',' + s.cell.c);

  const mul = Math.pow(1.5, cascadeLevel - 1);
  score += Math.floor(scoreThisStep * mul);
  updateScoreUI();
  const cells = [...clearing.values()];
  if (cells.length === 0 && spawns.length === 0) {
    inputLocked = false;
    return;
  }
  animateClear(cells, () => {
    for (const s of spawns) {
      board[s.cell.r][s.cell.c] = { color: s.color, special: s.special };
    }
    detonateSpecialsAndContinue(cells);
  });
}
```

(Note: `detonateSpecialsAndContinue` recursion ends by calling `applyGravityAndRefill` + `resolveCascades`. After cascadeLevel 1, no more power gems are spawned. ✓)

- [ ] **Step 4: Draw power-gem overlays**

In `draw()`, after drawing each gem image, if it has a `special`, draw an overlay:

```js
function drawSpecialOverlay(gem, x, y, size, now) {
  const cx = x + size / 2, cy = y + size / 2;
  if (gem.special === 'fire') {
    const pulse = 0.5 + 0.5 * Math.sin(now / 250);
    const grd = ctx.createRadialGradient(cx, cy, size * 0.15, cx, cy, size * 0.5);
    grd.addColorStop(0, `rgba(255,180,40,${0.0 + 0.5 * pulse})`);
    grd.addColorStop(1, 'rgba(255,80,0,0)');
    ctx.fillStyle = grd;
    ctx.fillRect(x, y, size, size);
  } else if (gem.special === 'star') {
    const rot = (now / 600) % (Math.PI * 2);
    ctx.save();
    ctx.translate(cx, cy);
    ctx.rotate(rot);
    ctx.strokeStyle = 'rgba(120,220,255,0.9)';
    ctx.lineWidth = 3;
    const arm = size * 0.45;
    ctx.beginPath();
    ctx.moveTo(-arm, 0); ctx.lineTo(arm, 0);
    ctx.moveTo(0, -arm); ctx.lineTo(0, arm);
    ctx.stroke();
    ctx.restore();
  } else if (gem.special === 'hyper') {
    // hyper gem's underlying image is the tile_hyper symbol — add hue-shift outline
    const hue = (now / 8) % 360;
    ctx.strokeStyle = `hsl(${hue}, 90%, 65%)`;
    ctx.lineWidth = 3;
    ctx.strokeRect(x + 2, y + 2, size - 4, size - 4);
  }
}

// in draw() gem loop, after ctx.drawImage(...):
if (g.special) drawSpecialOverlay(g, cx - size / 2, cy - size / 2, size, now);
```

Also: when `g.special === 'hyper'`, use `gemImages.hyper`; otherwise `gemImages[GEMS[g.color]]`. (Already wired in Task 2; just confirm the branch.) Fire and star use the colored gem's image with an overlay.

Since `drawSpecialOverlay` calls require `now`, ensure power gems always re-render — when any cell is a special, schedule a continuous redraw via `ensureAnimLoop` by pushing a sentinel "tick" animation, OR — simpler — start an ambient loop when `board` contains any special.

Add:

```js
function hasSpecials() {
  for (let r = 0; r < N; r++) for (let c = 0; c < N; c++) {
    if (board[r][c] && board[r][c].special) return true;
  }
  return false;
}
```

Modify the RAF tick so the loop keeps running while `hasSpecials()` is true (so the overlay animates).

```js
const tick = (now) => {
  let active = false;
  // ... handle anims as before ...
  draw();
  if (active || anims.length || hasSpecials()) {
    rafHandle = requestAnimationFrame(tick);
  } else {
    rafHandle = null;
    onAnimsSettled();
  }
};
```

And call `ensureAnimLoop()` whenever a power gem is spawned (already implicit if a clear anim runs first).

- [ ] **Step 5: Verify**

Open the page. Force a match-4 (you can use `preview_eval` to plant a board: e.g., set `board[3][0..3]` to color 0, surrounding rows to color 1, then swap into the line). After the swap, expect a fire gem to remain at the swap-origin cell, with a pulsing radial glow. Repeat for match-5 (star) and an L-shape (hyper). Swap the hyper with a colored gem: expect all gems of that color to clear in one animation.

---

## Task 6: Drop animation (smooth gravity)

**Files:**
- Modify: `index.html` (script block)

- [ ] **Step 1: Track drops instead of teleporting**

Replace `applyGravityAndRefill` to compute *moves* (from-row → to-row, per column, including new gems entering from above) and add per-cell drop animations.

```js
function applyGravityAndRefill() {
  const drops = [];  // { fromR, toR, c, newColor: null | int }
  for (let c = 0; c < N; c++) {
    let writeRow = N - 1;
    for (let r = N - 1; r >= 0; r--) {
      if (board[r][c]) {
        if (writeRow !== r) {
          drops.push({ fromR: r, toR: writeRow, c, gem: board[r][c] });
          board[writeRow][c] = board[r][c];
          board[r][c] = null;
        } else {
          // no drop needed
        }
        writeRow--;
      }
    }
    // fill remaining cells from above (fromR is negative = above board)
    let aboveOffset = -1;
    for (let r = writeRow; r >= 0; r--) {
      const gem = makeGem(Math.floor(Math.random() * GEMS.length));
      board[r][c] = gem;
      drops.push({ fromR: aboveOffset, toR: r, c, gem });
      aboveOffset--;
    }
  }
  // launch drop animation
  if (drops.length) {
    anims.push({
      kind: 'drop', t0: performance.now(),
      dur: 320, drops,
    });
    ensureAnimLoop();
  }
}
```

- [ ] **Step 2: Render drops**

```js
function dropOffsetFor(r, c, now) {
  for (const a of anims) {
    if (a.kind !== 'drop') continue;
    const t = Math.min(1, (now - a.t0) / a.dur);
    const e = easeOutCubic(t);
    for (const d of a.drops) {
      if (d.toR === r && d.c === c) {
        const startY = d.fromR * cellPx;
        const endY = d.toR * cellPx;
        const currentY = startY + (endY - startY) * e;
        return { dy: currentY - endY };
      }
    }
  }
  return { dy: 0 };
}

// in draw() gem loop, after gemRenderOffset:
const drop = dropOffsetFor(r, c, now);
const finalDy = off.dy + drop.dy;
// use finalDy instead of off.dy below
```

- [ ] **Step 3: Suppress draw of gems during their drop animation start frames?**

No — they're rendered with their offset; the new ones with negative `fromR` start above the board and slide down into place. Confirm via screenshot that gems fall instead of pop.

- [ ] **Step 4: Verify**

Make a match. Watch the gems above the cleared cells slide down, and new gems fall in from above the board's top edge. Cascade chains should still resolve correctly.

---

## Task 7: Deadlock detection + auto-shuffle + hint system

**Files:**
- Modify: `index.html` (script block)

- [ ] **Step 1: Find any legal swap**

```js
function findAnyLegalSwap() {
  for (let r = 0; r < N; r++) {
    for (let c = 0; c < N; c++) {
      // try right
      if (c + 1 < N) {
        swapCells(r, c, r, c + 1);
        const hasMatch = findMatches().length > 0;
        swapCells(r, c, r, c + 1);
        if (hasMatch) return { a: { r, c }, b: { r, c: c + 1 } };
      }
      // try down
      if (r + 1 < N) {
        swapCells(r, c, r + 1, c);
        const hasMatch = findMatches().length > 0;
        swapCells(r, c, r + 1, c);
        if (hasMatch) return { a: { r, c }, b: { r: r + 1, c } };
      }
    }
  }
  return null;
}
```

- [ ] **Step 2: Wire deadlock check + shuffle into `onAnimsSettled`**

```js
function shuffleBoard() {
  // collect colors+specials, shuffle, redistribute, retry until at least one legal swap
  for (let attempt = 0; attempt < 10; attempt++) {
    const pool = [];
    for (let r = 0; r < N; r++) for (let c = 0; c < N; c++) pool.push(board[r][c]);
    // Fisher-Yates
    for (let i = pool.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [pool[i], pool[j]] = [pool[j], pool[i]];
    }
    let k = 0;
    for (let r = 0; r < N; r++) for (let c = 0; c < N; c++) board[r][c] = pool[k++];
    if (findMatches().length === 0 && findAnyLegalSwap()) return true;
  }
  // last resort: rebuild
  newBoard();
  return true;
}

function showToast(msg, duration = 1200) {
  const t = document.getElementById('toast');
  t.textContent = msg;
  t.classList.add('show');
  setTimeout(() => t.classList.remove('show'), duration);
}

function onAnimsSettled() {
  if (inputLocked) return;
  if (!findAnyLegalSwap()) {
    showToast('Shuffling…', 1000);
    shuffleBoard();
    draw();
  } else {
    resetHintTimer();
  }
}
```

- [ ] **Step 3: Hint system**

```js
let hintTimer = null;
let hintCells = null;  // { a:{r,c}, b:{r,c} }

function resetHintTimer() {
  clearTimeout(hintTimer);
  hintCells = null;
  hintTimer = setTimeout(showHint, 5000);
}

function showHint() {
  hintCells = findAnyLegalSwap();
  if (hintCells) ensureAnimLoop();   // keep redrawing for the pulse
}

// in draw(), after gems:
if (hintCells) {
  const pulse = 0.5 + 0.5 * Math.sin(now / 250);
  ctx.strokeStyle = `rgba(255,216,74,${0.4 + 0.6 * pulse})`;
  ctx.lineWidth = 3;
  for (const cell of [hintCells.a, hintCells.b]) {
    ctx.strokeRect(cell.c * cellPx + 3, cell.r * cellPx + 3,
                   cellPx - 6, cellPx - 6);
  }
}
```

Also: in `RAF tick`, keep looping while `hintCells` is set.

```js
if (active || anims.length || hasSpecials() || hintCells) { /* schedule next frame */ }
```

Clear `hintCells` on any pointer input:

```js
canvas.addEventListener('pointerdown', (ev) => {
  resetHintTimer();
  // ... rest of handler
});
```

- [ ] **Step 4: Verify**

Open the page. Don't touch it for 5 seconds. Expect a soft yellow pulse on two adjacent gems that would form a match. Click anything → hint disappears, 5-second timer resets. To test deadlock, paint a known no-move position via `preview_eval` (e.g., a rainbow pattern with no possible match) and trigger `onAnimsSettled()`; expect "Shuffling…" toast + a refreshed board.

---

## Task 8: WebAudio sounds + mute toggle + best-score persistence

**Files:**
- Modify: `index.html` (script block)

- [ ] **Step 1: Audio module**

```js
let audioCtx = null;
let muted = false;

function ensureAudio() {
  if (audioCtx) return audioCtx;
  try { audioCtx = new (window.AudioContext || window.webkitAudioContext)(); }
  catch (e) { audioCtx = null; }
  return audioCtx;
}

function playBlip(freq, dur, type = 'sine', gain = 0.2) {
  if (muted) return;
  const ctx = ensureAudio();
  if (!ctx) return;
  const t = ctx.currentTime;
  const osc = ctx.createOscillator();
  const g = ctx.createGain();
  osc.type = type;
  osc.frequency.setValueAtTime(freq, t);
  g.gain.setValueAtTime(gain, t);
  g.gain.exponentialRampToValueAtTime(0.001, t + dur);
  osc.connect(g).connect(ctx.destination);
  osc.start(t);
  osc.stop(t + dur);
}

function playChord(freqs, dur, type = 'triangle', gain = 0.15) {
  for (const f of freqs) playBlip(f, dur, type, gain);
}

const SFX = {
  click: () => playBlip(800, 0.06, 'square', 0.1),
  swap:  () => { playBlip(500, 0.06, 'sine', 0.12);
                 setTimeout(() => playBlip(700, 0.08, 'sine', 0.12), 50); },
  match: (level = 1) => {
    const base = 440 * Math.pow(1.122, Math.min(level, 8));   // ~+2 semitones per cascade
    playChord([base, base * 1.25, base * 1.5], 0.25, 'triangle', 0.12);
  },
  bigMatch: () => playChord([523, 659, 784, 1047], 0.35, 'triangle', 0.14),
  power: () => {
    const ctx = ensureAudio(); if (!ctx || muted) return;
    const t = ctx.currentTime;
    const osc = ctx.createOscillator();
    const g = ctx.createGain();
    osc.type = 'sawtooth';
    osc.frequency.setValueAtTime(220, t);
    osc.frequency.exponentialRampToValueAtTime(880, t + 0.3);
    g.gain.setValueAtTime(0.15, t);
    g.gain.exponentialRampToValueAtTime(0.001, t + 0.3);
    osc.connect(g).connect(ctx.destination);
    osc.start(t); osc.stop(t + 0.3);
  },
  error: () => playBlip(180, 0.12, 'square', 0.1),
};
```

- [ ] **Step 2: Wire sounds into events**

- `attemptSwap`: on illegal → `SFX.error()`; on legal → `SFX.swap()`.
- `resolveCascades`: when matches found → `SFX.match(cascadeLevel)`; if any match length >= 4 → `SFX.bigMatch()` and `SFX.power()` if a power gem is spawned.
- `clearAndCascade` (hyper detonation): `SFX.power()`.
- Pointer-down on a gem (selecting): `SFX.click()`.

- [ ] **Step 3: Mute toggle + persistence**

```js
const LS_KEYS = { best: 'bejeweled.bestScore.v1', mute: 'bejeweled.mute.v1' };
let saveTimer = null;

function safeRead(k) { try { return localStorage.getItem(k); } catch (e) { return null; } }
function safeWrite(k, v) { try { localStorage.setItem(k, v); } catch (e) {} }

function loadPrefs() {
  const b = safeRead(LS_KEYS.best);
  if (b !== null) bestScore = parseInt(b, 10) || 0;
  const m = safeRead(LS_KEYS.mute);
  muted = m === '1';
}

function schedulePrefsSave() {
  clearTimeout(saveTimer);
  saveTimer = setTimeout(flushPrefs, 250);
}

function flushPrefs() {
  safeWrite(LS_KEYS.best, String(bestScore));
  safeWrite(LS_KEYS.mute, muted ? '1' : '0');
}

window.addEventListener('pagehide', flushPrefs);

let bestScore = 0;
function updateBest() {
  if (score > bestScore) {
    bestScore = score;
    document.getElementById('best').textContent = bestScore;
    schedulePrefsSave();
  }
}

// call updateBest() from updateScoreUI:
function updateScoreUI() {
  document.getElementById('score').textContent = score;
  updateBest();
}

document.getElementById('muteBtn').addEventListener('click', () => {
  muted = !muted;
  document.getElementById('muteBtn').textContent = muted ? '🔇' : '🔊';
  schedulePrefsSave();
});
```

Call `loadPrefs()` in boot, before `setupCanvas()`. Set initial mute button glyph and `#best` text from loaded state.

- [ ] **Step 4: Reset score on New Game**

```js
document.getElementById('newGameBtn').addEventListener('click', () => {
  document.getElementById('menuOverlay').classList.add('hidden');
  score = 0;
  updateScoreUI();
  newBoard();
  setupCanvas();
});
```

(Best score persists across games.)

- [ ] **Step 5: Verify**

Open page. Click around — expect short blips. Make a match — chord plays. Make a cascade — pitch rises with each chain. Hit mute button — silence + glyph changes. Reload — mute state and best score persist.

---

## Task 9: Menu polish + confirm-restart dialog + final UX pass

**Files:**
- Modify: `index.html` (script + DOM + CSS)

- [ ] **Step 1: Menu button shows confirm if mid-game**

```js
document.getElementById('menuBtn').addEventListener('click', () => {
  if (score > 0) {
    showDialog({
      title: 'New Game?',
      body: 'Your current score will be saved as your best if higher.',
      buttons: [
        { label: 'New Game', primary: true, onClick: () => {
            hideDialog();
            document.getElementById('menuOverlay').classList.remove('hidden');
        }},
        { label: 'Keep Playing', onClick: hideDialog },
      ],
    });
  } else {
    document.getElementById('menuOverlay').classList.remove('hidden');
  }
});

function showDialog({ title, body, buttons }) {
  const panel = document.getElementById('dialogPanel');
  panel.innerHTML = `<h1>${title}</h1><p style="opacity:0.8">${body || ''}</p>`;
  for (const b of buttons) {
    const btn = document.createElement('button');
    btn.textContent = b.label;
    if (!b.primary) btn.className = 'secondary';
    btn.onclick = b.onClick;
    panel.appendChild(btn);
  }
  document.getElementById('dialogOverlay').classList.remove('hidden');
}

function hideDialog() {
  document.getElementById('dialogOverlay').classList.add('hidden');
}
```

Also add `.panel p { margin: 0 0 16px; font-size: 14px; }` to CSS.

- [ ] **Step 2: Menu shows best score**

Update the panel HTML in `<body>`:

```html
<div class="panel">
  <h1>Bejeweled</h1>
  <p style="opacity:0.8; margin:0 0 16px">Match 3 or more gems.<br>
    4 = fire 💥 · 5 = star ✨ · L/T = hyper 🌈</p>
  <div style="margin:0 0 16px; opacity:0.7">Best: <span id="menuBest">0</span></div>
  <button id="newGameBtn">New Game</button>
</div>
```

After `loadPrefs()`, set `document.getElementById('menuBest').textContent = bestScore;`. Also update `menuBest` text when best changes.

- [ ] **Step 3: Disable double-tap zoom + visual scaling**

Confirm `<meta name="viewport">` includes `user-scalable=no`? No — we keep zoom for accessibility. Rely on `touch-action: none` on canvas to stop gesture interception.

- [ ] **Step 4: Verify desktop + mobile-emulated**

Use `preview_resize` to test 375×667 (iPhone SE) and 1024×768. Confirm:
- Board scales and stays centered with header above.
- Tap/swipe both work.
- Menu overlay covers full screen.
- Power-gem overlays animate.
- Score updates and persists.

---

## Task 10: PWA icons + manifest

**Files:**
- Create: `icon-master.svg`
- Create: `site.webmanifest`
- Create: `icon-192.png`, `icon-512.png`, `apple-touch-icon.png`, `apple-touch-icon-167x167.png`, `apple-touch-icon-152x152.png`, `favicon-16x16.png`, `favicon-32x32.png`, `favicon.ico`

- [ ] **Step 1: Author `icon-master.svg`**

A single stylized jewel on a dark rounded-square background, 1024×1024. Reuse the `tile_red` polygon design with the deep purple background of the game (#1a1230). Maskable-safe: keep the jewel inside a ~80% center safe area.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1024 1024">
  <rect width="1024" height="1024" rx="180" fill="#1a1230"/>
  <g transform="translate(512,512) scale(14)">
    <polygon points="15,20 20,15 20,-15 15,-20 -15,-20 -20,-15 -20,15 -15,20"
      fill="#f2212d" stroke="#ce0c17" stroke-width="2"/>
    <polygon points="15,20 20,15 20,-15 15,-20 -15,-20 -20,-15 -20,15 -15,20"
      fill="#f2212d" stroke="#ce0c17" stroke-width="4" transform="scale(0.5)"/>
  </g>
</svg>
```

- [ ] **Step 2: Check for ImageMagick**

```
magick --version
```

If absent, suggest installing it. Or use an online SVG→PNG service (skip for v1 if unavailable; the manifest still works without icons, but install prompts won't fire).

- [ ] **Step 3: Generate icons**

```
magick -background none icon-master.svg -resize 1024x1024 icon-512.png
magick -background none icon-master.svg -resize 192x192 icon-192.png
magick -background none icon-master.svg -resize 180x180 apple-touch-icon.png
magick -background none icon-master.svg -resize 167x167 apple-touch-icon-167x167.png
magick -background none icon-master.svg -resize 152x152 apple-touch-icon-152x152.png
magick -background none icon-master.svg -resize 32x32 favicon-32x32.png
magick -background none icon-master.svg -resize 16x16 favicon-16x16.png
magick -background none icon-master.svg -define icon:auto-resize=16,32,48 favicon.ico
```

- [ ] **Step 4: Write `site.webmanifest`**

```json
{
  "name": "Bejeweled",
  "short_name": "Bejeweled",
  "start_url": "./",
  "scope": "./",
  "display": "standalone",
  "orientation": "any",
  "background_color": "#1a1230",
  "theme_color": "#1a1230",
  "icons": [
    { "src": "icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "any maskable" },
    { "src": "icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any maskable" }
  ]
}
```

- [ ] **Step 5: Verify**

Open page locally. Check no 404 for manifest in `preview_network`. (PWA install prompt requires HTTPS, so it won't appear from `file://` — that's expected; verifying the manifest loads is enough.)

---

## Task 11: README

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write README**

Sections:
- Title, one-line description
- How to play (controls, special gems)
- Install on iPhone (Safari → Share → Add to Home Screen)
- Install on Android (Chrome → menu → Install app)
- Install on Desktop (address-bar install icon)
- File inventory
- Credits (gem SVG art from `bejeweled.html`)
- License (Unlicense by convention; ask user if they want different)

---

## Self-Review (run after writing this plan)

- **Spec coverage:** every spec section maps to tasks — sprite reuse (T1), board+render (T2), input+swap (T3), match+cascade (T4), power gems (T5), drop anim (T6), deadlock+hint (T7), audio+persistence (T8), menu polish (T9), PWA (T10), docs (T11). ✓
- **Placeholder scan:** no TBD/TODO; every step has code or a concrete command. ✓
- **Type consistency:** `board[r][c]` is `{color, special}` throughout. `anims` entries always have `kind`, `t0`, `dur`. `lastSwap` is `{a:{r,c}, b:{r,c}}` consistently. ✓
- **Naming consistency:** `attemptSwap`, `swapCells`, `findMatches`, `findAnyLegalSwap`, `resolveCascades`, `clearAndCascade`, `detonateSpecialsAndContinue`, `applyGravityAndRefill` all match between definition and call sites. ✓

Plan stands.
