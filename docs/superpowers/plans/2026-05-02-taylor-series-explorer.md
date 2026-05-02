# Taylor Series Explorer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Taylor series visualization page (`series.html`) with four tabs (A: polynomial stack, B: error bounds, C: convergence radius, D: Maclaurin/center). Implement A and B fully; scaffold C and D as "coming soon" placeholders. Add a third method card to the landing page.

**Architecture:** Single monolithic HTML file (`series.html`) following the existing `shell.html` / `washer.html` pattern. Three.js r160 via importmap CDN. `OrthographicCamera` looking down at xy plane (2D viz, but stays consistent with the rest of the codebase). Internal tab bar switches between four scene-builder functions that all add geometry to a shared disposable `stage` group.

**Tech Stack:** Vanilla JS, Three.js r160 via CDN importmap, no build step, no npm. Reuses `POSITIVE_GRADIENT` / `NEGATIVE_GRADIENT` color tokens and the `compileExpression` parser pattern from `shell.html`.

**Spec:** [docs/superpowers/specs/2026-05-02-taylor-series-explorer-design.md](../specs/2026-05-02-taylor-series-explorer-design.md) — read it before starting.

**Testing approach:** This project has no test framework (deliberate — no npm, no build step). Numeric code (derivative core, polynomial evaluator) gets inline `console.assert` blocks that run at page load and report PASS/FAIL via console. Visual code gets explicit visual-checkpoint steps with documented expected outputs. Do not introduce a test framework.

**Local dev:** `py -3.13 -m http.server 8765 --directory .` then open `http://localhost:8765/series.html`. The Claude Code preview tool sometimes hangs on the constantly-redrawing WebGL canvas; if that happens, opening the file in a real browser is the verification of record.

---

## File structure

**Files created:**
- `series.html` — entire page (~800-1000 lines, monolithic)
- `series-preview.png` — landing-page card screenshot of Tab A (created in Task 13)

**Files modified:**
- `index.html` — add third card; broaden title from "Volumes of Revolution" to a phrase that fits both volumes and series
- `CLAUDE.md` — add series-page architecture section

**Files unchanged:** `shell.html`, `washer.html`, `README.md`.

---

## Task 1: Scaffold series.html with chassis and tab bar

**Files:**
- Create: `series.html`

This task copies `washer.html` as the starting template, strips washer-specific 3D code, sets up the orthographic 2D scene, swaps the function library, and wires up the four-tab bar with placeholder bodies. After this task the page loads cleanly, you can switch tabs, and you can pick a function — but no math is drawn yet.

- [ ] **Step 1: Copy washer.html to series.html as starting template**

```bash
cp washer.html series.html
```

- [ ] **Step 2: Replace `<title>` and the header text in series.html**

Find:
```html
<title>Washer Method · 3D Explorer</title>
```
Replace with:
```html
<title>Taylor Series · 3D Explorer</title>
```

Find:
```html
<h1><b>Washer Method</b> · 3D Explorer</h1>
```
Replace with:
```html
<h1><b>Taylor Series</b> · Explorer</h1>
```

- [ ] **Step 3: Replace the preset pills in the `.presets` div**

Find the entire `<div class="presets">…</div>` block and replace with:
```html
<div class="presets">
  <button data-preset="sin" class="active">y = sin(x)</button>
  <button data-preset="cos">y = cos(x)</button>
  <button data-preset="exp">y = eˣ</button>
  <button data-preset="ln1p">y = ln(1+x)</button>
  <button data-preset="oneOver1Minus">y = 1/(1−x)</button>
  <button data-preset="oneOver1Plus">y = 1/(1+x²)</button>
  <button data-preset="custom">✏️ custom</button>
</div>
```

- [ ] **Step 4: Add the tab bar above the scene-wrap**

The tab bar is a new top-level child of `#app`. CSS grid currently has `grid-template-rows: auto auto 1fr auto` (header / customForm / scene-wrap / footer). Insert the tab bar as a new row between customForm and scene-wrap.

In the `<style>` section, find:
```css
#app { display: grid; grid-template-rows: auto auto 1fr auto; height: 100%; }
#app > header { grid-row: 1; }
#app > #customForm { grid-row: 2; }
#app > #scene-wrap { grid-row: 3; }
#app > footer { grid-row: 4; }
```

Replace with:
```css
#app { display: grid; grid-template-rows: auto auto auto 1fr auto; height: 100%; }
#app > header { grid-row: 1; }
#app > #customForm { grid-row: 2; }
#app > #tabBar { grid-row: 3; }
#app > #scene-wrap { grid-row: 4; }
#app > footer { grid-row: 5; }

#tabBar {
  display: flex; gap: 4px;
  padding: 8px 24px; border-bottom: 1px solid var(--pill-border);
  background: rgba(14, 17, 22, 0.6);
}
#tabBar button {
  background: transparent; color: var(--soft);
  border: none; border-bottom: 2px solid transparent;
  padding: 10px 18px;
  font-family: inherit; font-size: 13px;
  cursor: pointer; transition: all 0.15s;
  letter-spacing: 0.3px;
}
#tabBar button:hover { color: var(--ink); }
#tabBar button.active { color: var(--gold); border-bottom-color: var(--gold); }
#tabBar button .sub { color: var(--soft); margin-left: 8px; font-size: 11px; font-style: italic; }
#tabBar button.active .sub { color: var(--ink); }
```

In the body, find the `<div id="customForm">…</div>` block and add this directly after it (before `<div id="scene-wrap">`):
```html
<div id="tabBar">
  <button data-tab="A" class="active">A <span class="sub">building up</span></button>
  <button data-tab="B">B <span class="sub">error bounds</span></button>
  <button data-tab="C">C <span class="sub">convergence</span></button>
  <button data-tab="D">D <span class="sub">center point</span></button>
</div>
```

- [ ] **Step 5: Replace customForm with the series-appropriate version**

Find the entire `<div id="customForm">…</div>` block and replace with:
```html
<div id="customForm">
  <label>y =</label>
  <input id="customExpr" type="text" value="sin(x)" placeholder="e.g. exp(x) + sin(x)">
  <label>x ∈ [</label>
  <input id="customA" type="number" value="-6.28" step="0.1">
  <label>,</label>
  <input id="customB" type="number" value="6.28" step="0.1">
  <label>]</label>
  <button id="customApply">apply</button>
  <span id="customError"></span>
  <span id="customHelp">use <b>x</b>, <b>+ - * /</b>, <b>^</b> for power · functions: sin, cos, tan, sqrt, log, exp, abs · constants: pi, e</span>
</div>
```

- [ ] **Step 6: Update overlay to series-style content**

Find the `<div class="overlay">…</div>` block and replace its contents with:
```html
<div class="overlay">
  <h2><b id="ovTitle">y = sin(x)</b></h2>
  <div class="formula" id="ovFormula">Maclaurin series around a = 0</div>
  <div class="vol" id="ovTabReadout">Order n = 5</div>
  <div class="legend" id="ovLegend">
    <div><span class="swatch" style="background:#ffd166"></span>true f(x)</div>
    <div><span class="swatch" style="background:linear-gradient(90deg,#10204a,#6750d8,#d5cdff)"></span>polynomials P₀ … Pₙ (low → high order)</div>
    <div><span class="swatch" style="background:transparent;border:1px dashed #888"></span>center x = a</div>
  </div>
</div>
```

- [ ] **Step 7: Replace footer slider block with series-appropriate sliders**

Find the entire `<div class="controls">…</div>` block and replace with:
```html
<div class="controls">
  <div class="slider-block">
    <div class="label-row"><span>Order n</span><b id="lblOrder">5</b></div>
    <input id="orderSlider" type="range" min="0" max="20" value="5">
  </div>
  <div class="slider-block" id="centerBlock">
    <div class="label-row"><span>Center a</span><b id="lblCenter">0.00</b></div>
    <input id="centerSlider" type="range" min="-6.28" max="6.28" step="0.01" value="0">
  </div>
  <div class="slider-block" id="intervalBlock" style="display:none">
    <div class="label-row"><span>Interval</span><b id="lblInterval">[−1.5, 1.5]</b></div>
    <div style="display:flex; gap:6px;">
      <input id="intervalLoSlider" type="range" min="-6.28" max="0" step="0.01" value="-1.5" style="flex:1">
      <input id="intervalHiSlider" type="range" min="0" max="6.28" step="0.01" value="1.5" style="flex:1">
    </div>
  </div>
  <div class="button-stack">
    <button class="action" id="playBtn">▶  play</button>
    <select id="speedSel" class="speed-select" title="play speed">
      <option value="2.5">🐢 slow</option>
      <option value="1" selected>normal</option>
      <option value="0.4">🐇 fast</option>
      <option value="0.15">⚡ blitz</option>
    </select>
    <button class="action ghost" id="resetBtn">↺ reset view</button>
  </div>
</div>
<div class="hint" id="tabHint">drag the order slider — watch each polynomial creep outward from the center until it hugs the curve</div>
```

- [ ] **Step 8: Replace the entire `<script type="module">` block with a stub**

Find the existing module script (starts at the `import * as THREE` line) and replace the whole block with this stub. We're discarding all washer-specific JS:

```html
<script type="module">
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

// Same color tokens as shell.html and washer.html — do not diverge.
const POSITIVE_GRADIENT = [0x10204a, 0x6750d8, 0xd5cdff];
const NEGATIVE_GRADIENT = [0x4a1010, 0xc02a2a, 0xff7070];
const GOLD = 0xffd166;

// Function library. `f` is the function. `defaultA`, `defaultB` define the
// natural plotting window. `defaultCenter` is where Maclaurin/Taylor expands
// by default. `derivAt(x, k)` is the closed-form k-th derivative at x — for
// the named presets it's exact; the custom branch falls back to numeric
// finite differences (see Task 2).
const PRESETS = {
  sin: {
    label: 'y = sin(x)', formula: 'Maclaurin: x − x³/3! + x⁵/5! − …',
    f: Math.sin, defaultA: -2 * Math.PI, defaultB: 2 * Math.PI, defaultCenter: 0,
  },
  cos: {
    label: 'y = cos(x)', formula: 'Maclaurin: 1 − x²/2! + x⁴/4! − …',
    f: Math.cos, defaultA: -2 * Math.PI, defaultB: 2 * Math.PI, defaultCenter: 0,
  },
  exp: {
    label: 'y = eˣ', formula: 'Maclaurin: 1 + x + x²/2! + x³/3! + …',
    f: Math.exp, defaultA: -3, defaultB: 3, defaultCenter: 0,
  },
  ln1p: {
    label: 'y = ln(1+x)', formula: 'Maclaurin: x − x²/2 + x³/3 − …',
    f: x => Math.log(1 + x), defaultA: -0.95, defaultB: 3, defaultCenter: 0,
  },
  oneOver1Minus: {
    label: 'y = 1/(1−x)', formula: 'Maclaurin: 1 + x + x² + x³ + …',
    f: x => 1 / (1 - x), defaultA: -2, defaultB: 0.95, defaultCenter: 0,
  },
  oneOver1Plus: {
    label: 'y = 1/(1+x²)', formula: 'Maclaurin: 1 − x² + x⁴ − …',
    f: x => 1 / (1 + x * x), defaultA: -3, defaultB: 3, defaultCenter: 0,
  },
  custom: {
    label: 'custom', formula: 'enter your own function',
    f: Math.sin, defaultA: -2 * Math.PI, defaultB: 2 * Math.PI, defaultCenter: 0,
  },
};

// Mutable state.
let currentPreset = 'sin';
let currentTab = 'A';
let center = 0;          // a
let order = 5;           // n
let intervalLo = -1.5;   // Tab B only
let intervalHi = 1.5;    // Tab B only

// === Three.js setup ===
const canvas = document.getElementById('scene');
const renderer = new THREE.WebGLRenderer({ canvas, antialias: true, preserveDrawingBuffer: true });
renderer.setPixelRatio(window.devicePixelRatio);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x0e1116);

// Orthographic camera looking straight down at the xy plane. The "frustum"
// is sized in world units; the resize handler keeps the aspect square-true
// so circles don't get squashed.
const camera = new THREE.OrthographicCamera(-8, 8, 4, -4, 0.1, 100);
camera.position.set(0, 0, 10);
camera.up.set(0, 1, 0);
camera.lookAt(0, 0, 0);

const controls = new OrbitControls(camera, canvas);
controls.enableDamping = true;
controls.dampingFactor = 0.08;
controls.enableRotate = false;       // 2D — no rotation
controls.screenSpacePanning = true;
controls.target.set(0, 0, 0);

scene.add(new THREE.AmbientLight(0xffffff, 1.0));

const grid = new THREE.GridHelper(40, 40, 0x2a2f36, 0x1a1d22);
grid.rotation.x = Math.PI / 2;       // lay it flat in xy plane
scene.add(grid);

const stage = new THREE.Group();
scene.add(stage);

function disposeGroup(group) {
  group.traverse(obj => {
    if (obj.geometry) obj.geometry.dispose();
    if (obj.material) {
      if (Array.isArray(obj.material)) obj.material.forEach(m => m.dispose());
      else obj.material.dispose();
    }
  });
  group.clear();
}

function compileExpression(raw) {
  if (raw === undefined || raw === null) throw new Error('expression missing');
  let s = String(raw).trim().toLowerCase();
  if (!s) throw new Error('expression is empty');
  s = s.replace(/π/g, 'pi').replace(/\^/g, '**');
  if (/[=;]/.test(s)) throw new Error('use a single expression — no = or ;');
  let fn;
  try {
    fn = new Function('x', `with (Math) { return (${s}); }`);
  } catch (e) {
    throw new Error('syntax error — check your parentheses and operators');
  }
  for (const t of [-0.5, 0.01, 0.5, 1]) {
    const v = fn(t);
    if (typeof v !== 'number' || !isFinite(v)) {
      throw new Error(`function gives a non-number at x = ${t}`);
    }
  }
  return fn;
}

function gradientColor(stops, t) {
  t = Math.max(0, Math.min(1, t));
  const [c0, c1, c2] = stops.map(h => new THREE.Color(h));
  if (t <= 0.5) return new THREE.Color().lerpColors(c0, c1, t / 0.5);
  return new THREE.Color().lerpColors(c1, c2, (t - 0.5) / 0.5);
}

function rebuild() {
  disposeGroup(stage);
  // Each tab body adds geometry to `stage`. Filled in by later tasks.
  if (currentTab === 'A') buildTabA();
  else if (currentTab === 'B') buildTabB();
  else placeholderTab(currentTab);
  syncOverlay();
}

function placeholderTab(letter) {
  // Tabs C and D are spec'd but not built yet — leave the stage empty.
  // The overlay shows a "coming soon" message instead of math output.
}

function buildTabA() {
  // Filled in by Task 4.
}

function buildTabB() {
  // Filled in by Task 8.
}

function syncOverlay() {
  const p = PRESETS[currentPreset];
  document.getElementById('ovTitle').textContent = p.label;
  if (currentTab === 'C' || currentTab === 'D') {
    document.getElementById('ovFormula').textContent = `Tab ${currentTab}: coming soon — see design spec`;
    document.getElementById('ovTabReadout').textContent = '';
  } else {
    document.getElementById('ovFormula').textContent = p.formula;
    document.getElementById('ovTabReadout').textContent = `Order n = ${order}` +
      (currentTab === 'B' ? `   ·   interval [${intervalLo.toFixed(2)}, ${intervalHi.toFixed(2)}]` : '');
  }
}

// === Wire up controls ===
document.querySelectorAll('.presets button').forEach(btn => {
  btn.addEventListener('click', () => {
    document.querySelectorAll('.presets button').forEach(b => b.classList.remove('active'));
    btn.classList.add('active');
    currentPreset = btn.dataset.preset;
    document.getElementById('customForm').classList.toggle('shown', currentPreset === 'custom');
    rebuild();
  });
});

document.querySelectorAll('#tabBar button').forEach(btn => {
  btn.addEventListener('click', () => {
    document.querySelectorAll('#tabBar button').forEach(b => b.classList.remove('active'));
    btn.classList.add('active');
    currentTab = btn.dataset.tab;
    // Show interval slider only on Tab B; show center slider on A, B, D.
    document.getElementById('intervalBlock').style.display = currentTab === 'B' ? '' : 'none';
    document.getElementById('centerBlock').style.display = currentTab === 'C' ? 'none' : '';
    rebuild();
  });
});

document.getElementById('orderSlider').addEventListener('input', e => {
  order = parseInt(e.target.value, 10);
  document.getElementById('lblOrder').textContent = String(order);
  rebuild();
});

document.getElementById('centerSlider').addEventListener('input', e => {
  center = parseFloat(e.target.value);
  document.getElementById('lblCenter').textContent = center.toFixed(2);
  rebuild();
});

// === Resize handling ===
function resize() {
  const wrap = document.getElementById('scene-wrap');
  const w = wrap.clientWidth, h = wrap.clientHeight;
  renderer.setSize(w, h, false);
  // Maintain world-space units regardless of aspect: keep vertical span
  // fixed at 8 world units, derive horizontal span from aspect ratio.
  const vSpan = 8;
  const hSpan = vSpan * (w / h);
  camera.left = -hSpan / 2;
  camera.right = hSpan / 2;
  camera.top = vSpan / 2;
  camera.bottom = -vSpan / 2;
  camera.updateProjectionMatrix();
}
window.addEventListener('resize', resize);
resize();

// === Render loop ===
function tick() {
  controls.update();
  renderer.render(scene, camera);
  requestAnimationFrame(tick);
}
tick();

// Initial paint.
rebuild();
</script>
```

- [ ] **Step 9: Open in browser, confirm clean load**

Run the dev server (in a separate terminal — the Python http.server is the same one used by other pages):
```bash
py -3.13 -m http.server 8765 --directory .
```

Open `http://localhost:8765/series.html`. Expected:
- Page loads with no console errors
- Header reads "**Taylor Series** · Explorer"
- Function pills row shows 7 buttons
- Tab bar shows A / B / C / D, A active and gold-underlined
- Footer shows order slider (default 5) and center slider (default 0.00); interval block is hidden
- Scene area is empty grid (no curve drawn yet — that's Task 4)
- Clicking a function pill swaps the overlay title; clicking custom opens the customForm row
- Clicking a tab switches the gold underline; switching to B shows the interval block

Don't move on if any of the above is broken.

- [ ] **Step 10: Commit**

```bash
git add series.html
git commit -m "$(cat <<'EOF'
Scaffold series.html chassis

Copy washer.html, swap to OrthographicCamera + 2D layout, add four-tab bar,
swap function library to Taylor-friendly presets. Tabs A and B are wired
but render no math yet; tabs C and D are placeholders.

EOF
)"
```

---

## Task 2: Numeric derivative core

**Files:**
- Modify: `series.html` (add derivative module + inline tests)

The derivative core is the math engine for the whole page. For the six named presets we use closed-form derivative formulas (clean, no numeric noise, fast). For the custom branch we fall back to recursive 5-point finite differences (degrades past n≈10 but acceptable for visualization).

After this task the page console shows `[derivatives] all tests passed` on load.

- [ ] **Step 1: Add the derivative module to series.html**

Insert this block in the `<script type="module">` after the `PRESETS` declaration and before the mutable-state block (`let currentPreset = …`):

```javascript
// === Derivative core ===
// derivAt(presetKey, x, k) → k-th derivative of f at x.
// For named presets: closed-form. For custom: numeric finite difference.

const FACTORIAL = (() => {
  const f = [1];
  for (let i = 1; i <= 25; i++) f.push(f[i - 1] * i);
  return f;
})();

const PRESET_DERIVS = {
  sin(x, k) {
    // d^k/dx^k sin(x) = sin(x + kπ/2)
    const r = k % 4;
    if (r === 0) return Math.sin(x);
    if (r === 1) return Math.cos(x);
    if (r === 2) return -Math.sin(x);
    return -Math.cos(x);
  },
  cos(x, k) {
    // d^k/dx^k cos(x) = cos(x + kπ/2)
    const r = k % 4;
    if (r === 0) return Math.cos(x);
    if (r === 1) return -Math.sin(x);
    if (r === 2) return -Math.cos(x);
    return Math.sin(x);
  },
  exp(x, k) {
    return Math.exp(x);
  },
  ln1p(x, k) {
    if (k === 0) return Math.log(1 + x);
    // d^k/dx^k ln(1+x) = (-1)^(k-1) · (k-1)! / (1+x)^k
    const sign = (k - 1) % 2 === 0 ? 1 : -1;
    return sign * FACTORIAL[k - 1] / Math.pow(1 + x, k);
  },
  oneOver1Minus(x, k) {
    // d^k/dx^k 1/(1-x) = k! / (1-x)^(k+1)
    return FACTORIAL[k] / Math.pow(1 - x, k + 1);
  },
  oneOver1Plus(x, k) {
    // No closed-form clean enough — use numeric.
    return numericDerivative(PRESETS.oneOver1Plus.f, x, k);
  },
};

// Recursive finite difference. 5-point central stencil for first derivative,
// then recurse. This degrades past ~k=10 due to step-size noise; we cap
// the order slider at 20 and accept rougher estimates at the high end.
function numericDerivative(f, x, k, h = 1e-3) {
  if (k === 0) return f(x);
  if (k === 1) {
    // 5-point central: (-f(x+2h) + 8f(x+h) - 8f(x-h) + f(x-2h)) / (12h)
    return (-f(x + 2 * h) + 8 * f(x + h) - 8 * f(x - h) + f(x - 2 * h)) / (12 * h);
  }
  // For higher orders, recursively differentiate the (k-1)th derivative.
  // Using a wider step for higher k tames noise at the cost of resolution.
  const hk = Math.max(h, 1e-3 * Math.pow(1.4, k));
  const g = (xx) => numericDerivative(f, xx, k - 1, h);
  return (g(x + hk) - g(x - hk)) / (2 * hk);
}

function derivAt(presetKey, x, k) {
  const closed = PRESET_DERIVS[presetKey];
  if (closed) return closed(x, k);
  // Custom: use numeric on the compiled function stored in PRESETS.custom.f
  return numericDerivative(PRESETS[presetKey].f, x, k);
}

// === Self-tests for the derivative core ===
// Runs once at page load; logs PASS/FAIL to console. No test framework.
(function testDerivatives() {
  const cases = [
    // [name, actual, expected, tolerance]
    ['sin: f(0) = 0',         derivAt('sin', 0, 0), 0, 1e-9],
    ['sin: f\'(0) = 1',       derivAt('sin', 0, 1), 1, 1e-9],
    ['sin: f\'\'(0) = 0',     derivAt('sin', 0, 2), 0, 1e-9],
    ['sin: f\'\'\'(0) = -1',  derivAt('sin', 0, 3), -1, 1e-9],
    ['sin: f^(4)(0) = 0',     derivAt('sin', 0, 4), 0, 1e-9],
    ['cos: f(0) = 1',         derivAt('cos', 0, 0), 1, 1e-9],
    ['cos: f\'(0) = 0',       derivAt('cos', 0, 1), 0, 1e-9],
    ['cos: f\'\'(0) = -1',    derivAt('cos', 0, 2), -1, 1e-9],
    ['exp: f^(5)(0) = 1',     derivAt('exp', 0, 5), 1, 1e-9],
    ['exp: f^(3)(1) = e',     derivAt('exp', 1, 3), Math.E, 1e-9],
    ['ln1p: f(0) = 0',        derivAt('ln1p', 0, 0), 0, 1e-9],
    ['ln1p: f\'(0) = 1',      derivAt('ln1p', 0, 1), 1, 1e-9],
    ['ln1p: f\'\'(0) = -1',   derivAt('ln1p', 0, 2), -1, 1e-9],
    ['ln1p: f\'\'\'(0) = 2',  derivAt('ln1p', 0, 3), 2, 1e-9],
    ['1/(1-x): f(0) = 1',     derivAt('oneOver1Minus', 0, 0), 1, 1e-9],
    ['1/(1-x): f\'(0) = 1',   derivAt('oneOver1Minus', 0, 1), 1, 1e-9],
    ['1/(1-x): f\'\'(0) = 2', derivAt('oneOver1Minus', 0, 2), 2, 1e-9],
    ['1/(1-x): f\'\'\'(0)=6', derivAt('oneOver1Minus', 0, 3), 6, 1e-9],
    ['1/(1+x²): f(0) = 1',    derivAt('oneOver1Plus', 0, 0), 1, 1e-6],
    ['1/(1+x²): f\'(0) = 0',  derivAt('oneOver1Plus', 0, 1), 0, 1e-5],
    ['1/(1+x²): f\'\'(0)=-2', derivAt('oneOver1Plus', 0, 2), -2, 1e-3],
  ];
  let pass = 0, fail = 0;
  for (const [name, actual, expected, tol] of cases) {
    if (Math.abs(actual - expected) < tol) {
      pass++;
    } else {
      fail++;
      console.error(`[derivatives FAIL] ${name}: got ${actual}, expected ${expected}`);
    }
  }
  if (fail === 0) console.log(`[derivatives] all ${pass} tests passed`);
  else console.warn(`[derivatives] ${pass} passed, ${fail} FAILED`);
})();
```

- [ ] **Step 2: Reload series.html in browser, confirm tests pass**

Open the page, open dev tools console. Expected to see:
```
[derivatives] all 21 tests passed
```

If any FAIL lines appear, debug them before committing. The closed-form formulas in `PRESET_DERIVS` are the most likely source of bugs.

- [ ] **Step 3: Commit**

```bash
git add series.html
git commit -m "$(cat <<'EOF'
Add numeric derivative core with closed-form preset table

derivAt(presetKey, x, k) returns the k-th derivative. Six named presets use
hardcoded closed-form formulas; custom expressions and 1/(1+x²) fall back to
recursive 5-point finite differences. Inline self-tests run at page load and
log PASS/FAIL to console (no test framework).

EOF
)"
```

---

## Task 3: Polynomial evaluator with coefficient cache

**Files:**
- Modify: `series.html` (add polynomial module + inline tests)

`taylorPoly(coeffs, a, x)` evaluates the Taylor polynomial via Horner's method (numerically stable). `coefficientsAt(presetKey, a, n)` returns the array `[f(a), f'(a), f''(a)/2!, …, f^n(a)/n!]` and caches the result so polynomial evaluation across 400 plot points doesn't re-derive on every sample.

- [ ] **Step 1: Add the polynomial module to series.html**

Insert this block in the `<script type="module">` immediately after the derivative self-test IIFE:

```javascript
// === Polynomial evaluator + coefficient cache ===

// Horner-form evaluation: stable, O(n) per point.
// coeffs[k] = f^(k)(a) / k!  (i.e. already divided by factorial)
function taylorPoly(coeffs, a, x) {
  const dx = x - a;
  let result = coeffs[coeffs.length - 1];
  for (let i = coeffs.length - 2; i >= 0; i--) {
    result = result * dx + coeffs[i];
  }
  return result;
}

// Coefficient cache. Key = "presetKey|a|n". Cleared whenever the custom
// expression changes (the cache key includes 'custom' but the underlying
// f changes, so we explicitly clear in customApply).
const coeffCache = new Map();

function coefficientsAt(presetKey, a, n) {
  const key = `${presetKey}|${a.toFixed(6)}|${n}`;
  const hit = coeffCache.get(key);
  if (hit) return hit;
  const arr = new Array(n + 1);
  for (let k = 0; k <= n; k++) {
    arr[k] = derivAt(presetKey, a, k) / FACTORIAL[k];
  }
  coeffCache.set(key, arr);
  // Bound the cache so it doesn't grow unbounded across sessions of dragging.
  if (coeffCache.size > 500) {
    const firstKey = coeffCache.keys().next().value;
    coeffCache.delete(firstKey);
  }
  return arr;
}

// === Self-tests ===
(function testTaylor() {
  const cases = [];

  // sin(x) Maclaurin to order 5: 0 + 1·x + 0 - x³/6 + 0 + x⁵/120
  // Evaluated at x = 0.5 should match sin(0.5) ≈ 0.4794 to ~4 decimal places.
  const sinCoeffs = coefficientsAt('sin', 0, 5);
  cases.push(['sin Taylor coeff[0] = 0', sinCoeffs[0], 0, 1e-9]);
  cases.push(['sin Taylor coeff[1] = 1', sinCoeffs[1], 1, 1e-9]);
  cases.push(['sin Taylor coeff[3] = -1/6', sinCoeffs[3], -1/6, 1e-9]);
  cases.push(['sin Taylor coeff[5] = 1/120', sinCoeffs[5], 1/120, 1e-9]);
  cases.push(['sin P5(0.5) ≈ sin(0.5)', taylorPoly(sinCoeffs, 0, 0.5), Math.sin(0.5), 1e-4]);

  // exp(x) Taylor around a=1, evaluated at x=1 should be exactly e.
  const expCoeffs = coefficientsAt('exp', 1, 10);
  cases.push(['exp P10(1) at a=1 = e', taylorPoly(expCoeffs, 1, 1), Math.E, 1e-9]);
  cases.push(['exp P10(1.5) at a=1 ≈ exp(1.5)',
              taylorPoly(expCoeffs, 1, 1.5), Math.exp(1.5), 1e-4]);

  // 1/(1-x) Maclaurin coefficients should all be 1.
  const geomCoeffs = coefficientsAt('oneOver1Minus', 0, 8);
  cases.push(['1/(1-x) coeff[0] = 1', geomCoeffs[0], 1, 1e-9]);
  cases.push(['1/(1-x) coeff[5] = 1', geomCoeffs[5], 1, 1e-9]);
  cases.push(['1/(1-x) P8(0.5) ≈ 2', taylorPoly(geomCoeffs, 0, 0.5), 1 / (1 - 0.5), 1e-2]);

  // Cache hit returns the same array reference.
  const a = coefficientsAt('sin', 0, 5);
  const b = coefficientsAt('sin', 0, 5);
  cases.push(['cache returns same array', a === b ? 1 : 0, 1, 0]);

  let pass = 0, fail = 0;
  for (const [name, actual, expected, tol] of cases) {
    if (Math.abs(actual - expected) <= tol) {
      pass++;
    } else {
      fail++;
      console.error(`[taylor FAIL] ${name}: got ${actual}, expected ${expected}`);
    }
  }
  if (fail === 0) console.log(`[taylor] all ${pass} tests passed`);
  else console.warn(`[taylor] ${pass} passed, ${fail} FAILED`);
})();
```

- [ ] **Step 2: Reload, confirm both test blocks pass**

Console should now show two PASS lines:
```
[derivatives] all 21 tests passed
[taylor] all 11 tests passed
```

- [ ] **Step 3: Commit**

```bash
git add series.html
git commit -m "$(cat <<'EOF'
Add Taylor polynomial evaluator with coefficient cache

taylorPoly uses Horner's method for stability. coefficientsAt caches by
(preset, a, n) so the 400-sample plot loop reuses derivatives instead of
recomputing per point. Inline tests verify sin/exp/1/(1-x) Maclaurin and
Taylor expansions, plus cache identity.

EOF
)"
```

---

## Task 4: Tab A scene rendering

**Files:**
- Modify: `series.html` (replace `buildTabA()` stub with real implementation)

This task draws the gold f(x) curve, the polynomial stack P₀…Pₙ, the dashed center line, and the anchor dot. After this task Tab A is visually complete.

Three.js note: we use `THREE.Line` with `LineBasicMaterial` for the curves rather than `TubeGeometry`. Tubes look great in 3D but don't render well in the orthographic 2D view (no thickness illusion from perspective). `Line` with `linewidth: 1` is fine and 10× cheaper.

- [ ] **Step 1: Add curve-builder helpers and Tab A implementation**

Replace the `function buildTabA() { /* Filled in by Task 4. */ }` stub with the following block. Keep `buildTabB()` and `placeholderTab()` as-is.

```javascript
// === Plotting helpers ===

const PLOT_SAMPLES = 400;

// Build a Line object from sampled (x, y) pairs in the xy plane (z=0).
// Skips segments where y is non-finite (lets us safely plot 1/(1-x) etc.)
function buildPolyline(xs, ys, color, opacity = 1.0) {
  const points = [];
  for (let i = 0; i < xs.length; i++) {
    const y = ys[i];
    if (!isFinite(y)) {
      // Push the previous segment as its own line by ending here, then start fresh.
      // For simplicity we just skip; gaps are visually fine.
      continue;
    }
    points.push(new THREE.Vector3(xs[i], y, 0));
  }
  const geom = new THREE.BufferGeometry().setFromPoints(points);
  const mat = new THREE.LineBasicMaterial({ color, transparent: opacity < 1, opacity });
  return new THREE.Line(geom, mat);
}

// Sample the visible x-window into PLOT_SAMPLES evenly spaced xs.
function sampleX(p) {
  const xs = new Array(PLOT_SAMPLES);
  for (let i = 0; i < PLOT_SAMPLES; i++) {
    xs[i] = p.defaultA + (p.defaultB - p.defaultA) * (i / (PLOT_SAMPLES - 1));
  }
  return xs;
}

// Y-axis clipping. High-order polynomials shoot to ±∞; without clipping
// they'd crush the rest of the scene to a flat strip. Clip anything outside
// [yMin, yMax] to ±10× the visible y range so the line visibly slams into
// the edge rather than disappearing.
function clipY(ys, yMin, yMax) {
  const span = yMax - yMin;
  const lo = yMin - 0.1 * span;
  const hi = yMax + 0.1 * span;
  return ys.map(y => {
    if (!isFinite(y)) return NaN;
    if (y > hi) return hi;
    if (y < lo) return lo;
    return y;
  });
}

function buildTabA() {
  const p = PRESETS[currentPreset];
  const xs = sampleX(p);

  // True curve: gold line.
  const trueYs = xs.map(p.f);
  const yMin = Math.min(...trueYs.filter(isFinite));
  const yMax = Math.max(...trueYs.filter(isFinite));
  const trueClipped = clipY(trueYs, yMin, yMax);
  stage.add(buildPolyline(xs, trueClipped, GOLD, 1.0));

  // Polynomial stack P_0 … P_n. Color from POSITIVE_GRADIENT, low order = darkest.
  for (let k = 0; k <= order; k++) {
    const coeffs = coefficientsAt(currentPreset, center, k);
    const polyYs = xs.map(x => taylorPoly(coeffs, center, x));
    const polyClipped = clipY(polyYs, yMin, yMax);
    const t = order === 0 ? 1.0 : k / order;
    const color = gradientColor(POSITIVE_GRADIENT, t);
    // Higher-order polynomials drawn slightly more opaque so they pop above the stack.
    const opacity = 0.55 + 0.45 * t;
    stage.add(buildPolyline(xs, polyClipped, color, opacity));
  }

  // Dashed vertical line at x = center.
  const centerLineGeom = new THREE.BufferGeometry().setFromPoints([
    new THREE.Vector3(center, yMin - 0.5, 0),
    new THREE.Vector3(center, yMax + 0.5, 0),
  ]);
  const centerLineMat = new THREE.LineDashedMaterial({
    color: 0x888888, dashSize: 0.15, gapSize: 0.1, opacity: 0.6, transparent: true,
  });
  const centerLine = new THREE.Line(centerLineGeom, centerLineMat);
  centerLine.computeLineDistances();
  stage.add(centerLine);

  // Anchor dot at (center, f(center)).
  const dotGeom = new THREE.CircleGeometry(0.07, 24);
  const dotMat = new THREE.MeshBasicMaterial({ color: GOLD });
  const dot = new THREE.Mesh(dotGeom, dotMat);
  dot.position.set(center, p.f(center), 0.01);
  stage.add(dot);

  // Auto-fit the camera y-window to [yMin, yMax] with margin.
  fitCameraY(yMin, yMax, p.defaultA, p.defaultB);
}

// Adjust the orthographic camera so [a, b] fits horizontally with margin
// and [yMin, yMax] fits vertically with margin. Maintains aspect.
function fitCameraY(yMin, yMax, xMin, xMax) {
  const xPad = (xMax - xMin) * 0.05;
  const yPad = (yMax - yMin) * 0.15 + 0.5;
  const wrap = document.getElementById('scene-wrap');
  const aspect = wrap.clientWidth / wrap.clientHeight;
  const ySpan = (yMax - yMin) + 2 * yPad;
  const xSpan = (xMax - xMin) + 2 * xPad;
  // Pick the larger required span; the other dimension fills via aspect.
  const vSpan = Math.max(ySpan, xSpan / aspect);
  const yCenter = (yMin + yMax) / 2;
  const xCenter = (xMin + xMax) / 2;
  camera.left = xCenter - (vSpan * aspect) / 2;
  camera.right = xCenter + (vSpan * aspect) / 2;
  camera.top = yCenter + vSpan / 2;
  camera.bottom = yCenter - vSpan / 2;
  camera.updateProjectionMatrix();
  controls.target.set(xCenter, yCenter, 0);
}
```

- [ ] **Step 2: Reload series.html, visual checkpoint**

Reload the page (default state: `sin(x)`, Tab A, n=5, a=0). Expected:
- Gold sinusoid drawn over `[−2π, 2π]`, oscillating between roughly ±1
- Stack of 6 polynomial curves (P₀ through P₅), darkest blue at the bottom of the gradient (P₀, the constant 0), palest lavender for P₅ (which should hug the gold curve well in the middle but diverge near the edges)
- Dashed gray vertical line at x=0
- Small gold dot at (0, 0)

Drag the order slider to 0: only P₀ visible (flat line at y=0). Drag to 20: stack of 21 polynomials, all hugging the gold curve closely across the visible window.

Click `y = exp(x)` pill: gold curve becomes exponential, polynomial stack rebuilds and fans out from origin. Click `y = 1/(1-x)`: gold curve has visible asymptote behavior near x=1; polynomials hug it inside [-1, 1] and shoot off outside (clipping should kick in).

- [ ] **Step 3: Commit**

```bash
git add series.html
git commit -m "$(cat <<'EOF'
Tab A: render polynomial stack converging on f(x)

Gold true curve + P_0 through P_n stacked, color-ramped by order via
POSITIVE_GRADIENT. Dashed center line and anchor dot. Y-axis clipping so
high-order polynomials slam into viewport edges instead of crushing the
scene. Camera auto-fits the function's natural window.

EOF
)"
```

---

## Task 5: Tab A controls — play, formula readout, custom expression

**Files:**
- Modify: `series.html`

Wire up the play button to animate `n` from 0 to its current value, the speed selector to control animation rate, the reset button to recenter the camera, and the custom-expression apply button to recompile + re-render. Add a polynomial formula readout in the overlay that updates as `n` and `a` change.

- [ ] **Step 1: Add play / reset / custom-apply / formula-readout logic**

Insert this block in the `<script type="module">` after the `centerSlider` event listener and before the `resize` function:

```javascript
// === Play / reset / custom apply / formula readout ===

let playTimer = null;

function stopPlay() {
  if (playTimer) {
    clearInterval(playTimer);
    playTimer = null;
    document.getElementById('playBtn').textContent = '▶  play';
  }
}

document.getElementById('playBtn').addEventListener('click', () => {
  if (playTimer) { stopPlay(); return; }
  if (currentTab !== 'A') return;  // only Tab A has play for now
  document.getElementById('playBtn').textContent = '⏸  pause';
  const speed = parseFloat(document.getElementById('speedSel').value);
  const interval = Math.max(60, 220 * speed);
  let n = 0;
  const maxN = parseInt(document.getElementById('orderSlider').max, 10);
  playTimer = setInterval(() => {
    order = n;
    document.getElementById('orderSlider').value = String(n);
    document.getElementById('lblOrder').textContent = String(n);
    rebuild();
    n++;
    if (n > maxN) stopPlay();
  }, interval);
});

document.getElementById('resetBtn').addEventListener('click', () => {
  stopPlay();
  const p = PRESETS[currentPreset];
  center = p.defaultCenter;
  document.getElementById('centerSlider').value = String(center);
  document.getElementById('lblCenter').textContent = center.toFixed(2);
  rebuild();
});

document.getElementById('customApply').addEventListener('click', () => {
  const expr = document.getElementById('customExpr').value;
  const aVal = parseFloat(document.getElementById('customA').value);
  const bVal = parseFloat(document.getElementById('customB').value);
  const errEl = document.getElementById('customError');
  errEl.textContent = '';
  try {
    const fn = compileExpression(expr);
    PRESETS.custom.f = fn;
    PRESETS.custom.label = `y = ${expr}`;
    PRESETS.custom.formula = `custom expression around a = ${center.toFixed(2)}`;
    PRESETS.custom.defaultA = aVal;
    PRESETS.custom.defaultB = bVal;
    coeffCache.clear();
    if (currentPreset !== 'custom') {
      // Switch to custom preset on apply.
      document.querySelectorAll('.presets button').forEach(b => b.classList.remove('active'));
      document.querySelector('.presets button[data-preset="custom"]').classList.add('active');
      currentPreset = 'custom';
    }
    rebuild();
  } catch (e) {
    errEl.textContent = e.message;
  }
});

// Formula readout: shows P_n(x) with numeric coefficients filled in.
// Example for sin(x) Maclaurin n=5:
//   P5(x) = 0 + 1·x − 0.1667·x³ + 0.00833·x⁵
function formatPolynomialReadout(coeffs, a) {
  const terms = [];
  for (let k = 0; k < coeffs.length; k++) {
    const c = coeffs[k];
    if (Math.abs(c) < 1e-9) continue;
    let coef;
    if (k === 0) coef = c.toFixed(4);
    else coef = (Math.abs(c) === 1 ? '' : Math.abs(c).toFixed(4)) + '·';
    const sign = terms.length === 0 ? (c < 0 ? '−' : '') : (c < 0 ? ' − ' : ' + ');
    let xPart;
    if (k === 0) xPart = '';
    else if (a === 0) xPart = k === 1 ? 'x' : `x^${k}`;
    else {
      const aStr = a >= 0 ? `(x − ${a.toFixed(2)})` : `(x + ${(-a).toFixed(2)})`;
      xPart = k === 1 ? aStr : `${aStr}^${k}`;
    }
    terms.push(sign + coef + xPart);
  }
  if (terms.length === 0) return '0';
  return `P${coeffs.length - 1}(x) = ${terms.join('')}`;
}
```

- [ ] **Step 2: Update `syncOverlay` to include the polynomial readout for Tab A**

Find the `syncOverlay` function and replace it with:

```javascript
function syncOverlay() {
  const p = PRESETS[currentPreset];
  document.getElementById('ovTitle').textContent = p.label;
  if (currentTab === 'C' || currentTab === 'D') {
    document.getElementById('ovFormula').textContent = `Tab ${currentTab}: coming soon — see design spec`;
    document.getElementById('ovTabReadout').textContent = '';
    return;
  }
  document.getElementById('ovFormula').textContent = p.formula;
  if (currentTab === 'A') {
    const coeffs = coefficientsAt(currentPreset, center, order);
    document.getElementById('ovTabReadout').textContent = formatPolynomialReadout(coeffs, center);
  } else if (currentTab === 'B') {
    // Tab B readout filled in by Task 11.
    document.getElementById('ovTabReadout').textContent =
      `n = ${order}   ·   interval [${intervalLo.toFixed(2)}, ${intervalHi.toFixed(2)}]`;
  }
}
```

- [ ] **Step 3: Visual checkpoint**

Reload. Default state: `sin(x)`, Tab A, n=5, a=0. Overlay should show:
```
y = sin(x)
Maclaurin: x − x³/3! + x⁵/5! − …
P5(x) = 1.0000·x − 0.1667·x^3 + 0.0083·x^5
```

Click ▶ play. The polynomial stack should grow from 1 polynomial up to 21, one new polynomial added per ~220ms (at normal speed). The order slider knob should advance in lockstep, and the formula readout should update. Click again to pause.

Drag the center slider to ~1.5. The polynomial stack should re-anchor at x=1.5; the dashed line and gold dot should follow; the formula readout should switch to `P5(x) = … (x − 1.50) …` form.

Click `✏️ custom`, enter `cos(x) - 0.5`, click apply. Should re-render with the new function. Try a malformed expression (e.g. `sin(x` missing paren) and verify the error message appears in red.

Click ↺ reset view. Should snap center back to 0.

- [ ] **Step 4: Commit**

```bash
git add series.html
git commit -m "$(cat <<'EOF'
Tab A: play button, custom expression, live formula readout

Play animates n from 0 upward at the chosen speed. Custom-apply recompiles
the expression and clears the coefficient cache. Overlay shows the current
polynomial in Horner-readable form with numeric coefficients, updating as
order or center change.

EOF
)"
```

---

## Task 6: Visual polish + landing-card-ready Tab A

**Files:**
- Modify: `series.html`

Tab A is functional but needs a visual pass before we screenshot it for the landing card. This task: improve line widths, fade the polynomial stack outside the convergence-friendly window, and make the anchor dot and center line read better.

- [ ] **Step 1: Improve `LineBasicMaterial` thickness perception**

`linewidth` is ignored on most platforms — Three.js `Line` always renders 1px regardless. To make the gold curve and the active polynomial visually pop, draw them twice: once with full opacity and once with a translucent halo at slightly different y-offset (z-offset is invisible in orthographic top-down view).

In `buildTabA`, after `stage.add(buildPolyline(xs, trueClipped, GOLD, 1.0));`, add a second halo line. Replace that single line with:

```javascript
  // True curve: gold halo + crisp inner line.
  const haloMat = new THREE.LineBasicMaterial({ color: GOLD, transparent: true, opacity: 0.25 });
  const haloPoints = [];
  for (let i = 0; i < xs.length; i++) {
    const y = trueClipped[i];
    if (!isFinite(y)) continue;
    haloPoints.push(new THREE.Vector3(xs[i], y, -0.005));
  }
  const haloGeom = new THREE.BufferGeometry().setFromPoints(haloPoints);
  stage.add(new THREE.Line(haloGeom, haloMat));
  stage.add(buildPolyline(xs, trueClipped, GOLD, 1.0));
```

(The halo at slight negative z renders behind the inner line because the orthographic camera looks down the +z axis from z=10.)

- [ ] **Step 2: Make the highest-order polynomial stand out**

Inside the polynomial stack loop, when `k === order` (the highest), give it a larger anchor dot and slightly thicker halo. Replace the loop body inside `buildTabA` with:

```javascript
  for (let k = 0; k <= order; k++) {
    const coeffs = coefficientsAt(currentPreset, center, k);
    const polyYs = xs.map(x => taylorPoly(coeffs, center, x));
    const polyClipped = clipY(polyYs, yMin, yMax);
    const t = order === 0 ? 1.0 : k / order;
    const color = gradientColor(POSITIVE_GRADIENT, t);
    if (k === order && order > 0) {
      // Halo for the active (highest-order) polynomial.
      const lavenderHaloMat = new THREE.LineBasicMaterial({ color, transparent: true, opacity: 0.25 });
      const lavenderHaloPoints = [];
      for (let i = 0; i < xs.length; i++) {
        const y = polyClipped[i];
        if (!isFinite(y)) continue;
        lavenderHaloPoints.push(new THREE.Vector3(xs[i], y, -0.004));
      }
      const lavenderHaloGeom = new THREE.BufferGeometry().setFromPoints(lavenderHaloPoints);
      stage.add(new THREE.Line(lavenderHaloGeom, lavenderHaloMat));
    }
    const opacity = 0.5 + 0.5 * t;
    stage.add(buildPolyline(xs, polyClipped, color, opacity));
  }
```

- [ ] **Step 3: Visual checkpoint**

Reload. The gold curve and the highest-order polynomial should both have a soft halo around them. Lower-order polynomials in the middle of the stack should be visibly less prominent. The whole composition should look like a "fan" of polynomial curves with the gold reference and the leading polynomial standing out.

Try `1/(1+x²)` with n=8. The Maclaurin series should hug the curve inside [-1, 1] and slam into the viewport edges outside (the convergence-radius behavior — Tab C will make this explicit later, but it's already visible).

- [ ] **Step 4: Commit**

```bash
git add series.html
git commit -m "$(cat <<'EOF'
Tab A: visual polish — halos on gold curve and active polynomial

Three.js Line ignores linewidth on most platforms, so add translucent halo
lines at slight z-offsets to fake stroke weight. Makes the true curve and
the leading polynomial pop above the rest of the stack.

EOF
)"
```

---

## Task 7: Tab A checkpoint — verify chassis carries weight

**Files:** none (verification-only task)

This is the chassis checkpoint from the spec. Before building Tab B, confirm Tab A actually works end-to-end across all named presets. If the chassis doesn't carry weight here, fix it before moving on — Tab B will inherit the same architecture.

- [ ] **Step 1: Run through all six named presets at multiple n values**

For each preset below, set `n=5` then `n=15` and observe:

| Preset | Expected |
|---|---|
| `sin(x)` | Polynomials hug the curve in middle, diverge near ±2π. |
| `cos(x)` | Similar to sin. P₀ is constant 1. |
| `eˣ` | Polynomials underestimate then catch up; high n hugs across the window. |
| `ln(1+x)` | Polynomials work in [−0.95, 1] roughly; diverge sharply outside. |
| `1/(1−x)` | Polynomials hug inside [−1, 1], explode for x > 1. |
| `1/(1+x²)` | Same convergence-radius story as 1/(1−x), but happens at x = ±1. |

- [ ] **Step 2: Drag center slider on `sin(x)` from −π to π**

The polynomial stack should smoothly re-anchor as you drag. Anchor dot stays on the curve. No console errors, no flickering, no NaN points causing line gaps.

- [ ] **Step 3: Custom expression sanity test**

Click custom, enter `x*x*x - 2*x` (a cubic), apply with x ∈ [−2, 2]. The Taylor series of a cubic IS the cubic itself, so all polynomials of order ≥ 3 should overlap the gold curve exactly. Drag center slider: should still be exact (it's just the same cubic re-expressed around a different center).

- [ ] **Step 4: If anything broken, fix it now and commit before moving on**

If everything works, no commit needed for this task — it's a verification gate.

---

## Task 8: Tab B — single polynomial + error band

**Files:**
- Modify: `series.html` (replace `buildTabB()` stub)

Tab B shows a single polynomial Pₙ in pale lavender plus the actual signed error `f(x) − Pₙ(x)` rendered as a translucent red band (filled mesh between f and Pₙ). The Lagrange envelope and interval handles come in Tasks 9 and 10.

- [ ] **Step 1: Add error-band geometry helper**

Insert this helper in the script, right after `buildPolyline`:

```javascript
// Triangle-strip mesh between two polylines (top = ys1, bottom = ys2),
// in the xy plane. Used for the error band in Tab B.
function buildBand(xs, ysTop, ysBottom, color, opacity) {
  // Build a vertex array of paired top/bottom points, indexed as a triangle strip.
  const vertices = [];
  const indices = [];
  let idx = 0;
  for (let i = 0; i < xs.length; i++) {
    const yt = ysTop[i], yb = ysBottom[i];
    if (!isFinite(yt) || !isFinite(yb)) continue;
    vertices.push(xs[i], yt, 0, xs[i], yb, 0);
    if (idx > 0) {
      // Two triangles: (prev_top, prev_bot, curr_top) and (curr_top, prev_bot, curr_bot)
      const pT = (idx - 1) * 2, pB = pT + 1, cT = idx * 2, cB = cT + 1;
      indices.push(pT, pB, cT, cT, pB, cB);
    }
    idx++;
  }
  const geom = new THREE.BufferGeometry();
  geom.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3));
  geom.setIndex(indices);
  const mat = new THREE.MeshBasicMaterial({
    color, transparent: true, opacity, side: THREE.DoubleSide, depthWrite: false,
  });
  return new THREE.Mesh(geom, mat);
}
```

- [ ] **Step 2: Implement `buildTabB`**

Replace the `function buildTabB() { /* Filled in by Task 8. */ }` stub with:

```javascript
function buildTabB() {
  const p = PRESETS[currentPreset];
  const xs = sampleX(p);

  const trueYs = xs.map(p.f);
  const yMin = Math.min(...trueYs.filter(isFinite));
  const yMax = Math.max(...trueYs.filter(isFinite));
  const trueClipped = clipY(trueYs, yMin, yMax);

  // Single polynomial P_n in pale lavender (top of POSITIVE_GRADIENT).
  const coeffs = coefficientsAt(currentPreset, center, order);
  const polyYs = xs.map(x => taylorPoly(coeffs, center, x));
  const polyClipped = clipY(polyYs, yMin, yMax);

  // Error band BETWEEN trueClipped and polyClipped.
  // Color from NEGATIVE_GRADIENT — we want red for "this is the error".
  const errColor = new THREE.Color(NEGATIVE_GRADIENT[1]);
  stage.add(buildBand(xs, trueClipped, polyClipped, errColor, 0.30));

  // Halo + crisp gold true curve.
  const haloMat = new THREE.LineBasicMaterial({ color: GOLD, transparent: true, opacity: 0.25 });
  const haloPoints = [];
  for (let i = 0; i < xs.length; i++) {
    if (!isFinite(trueClipped[i])) continue;
    haloPoints.push(new THREE.Vector3(xs[i], trueClipped[i], -0.005));
  }
  stage.add(new THREE.Line(new THREE.BufferGeometry().setFromPoints(haloPoints), haloMat));
  stage.add(buildPolyline(xs, trueClipped, GOLD, 1.0));

  // Single lavender polynomial.
  const lavender = gradientColor(POSITIVE_GRADIENT, 1.0);
  stage.add(buildPolyline(xs, polyClipped, lavender, 1.0));

  // Dashed center line + anchor dot (same as Tab A).
  const centerLineGeom = new THREE.BufferGeometry().setFromPoints([
    new THREE.Vector3(center, yMin - 0.5, 0),
    new THREE.Vector3(center, yMax + 0.5, 0),
  ]);
  const centerLineMat = new THREE.LineDashedMaterial({
    color: 0x888888, dashSize: 0.15, gapSize: 0.1, opacity: 0.6, transparent: true,
  });
  const centerLine = new THREE.Line(centerLineGeom, centerLineMat);
  centerLine.computeLineDistances();
  stage.add(centerLine);

  const dot = new THREE.Mesh(
    new THREE.CircleGeometry(0.07, 24),
    new THREE.MeshBasicMaterial({ color: GOLD })
  );
  dot.position.set(center, p.f(center), 0.01);
  stage.add(dot);

  fitCameraY(yMin, yMax, p.defaultA, p.defaultB);
}
```

- [ ] **Step 3: Update overlay legend when on Tab B**

In `syncOverlay`, before the `if (currentTab === 'C' || …)` line, add legend swapping:

```javascript
function syncOverlay() {
  const p = PRESETS[currentPreset];
  document.getElementById('ovTitle').textContent = p.label;

  // Swap legend per tab.
  const legend = document.getElementById('ovLegend');
  if (currentTab === 'A') {
    legend.innerHTML = `
      <div><span class="swatch" style="background:#ffd166"></span>true f(x)</div>
      <div><span class="swatch" style="background:linear-gradient(90deg,#10204a,#6750d8,#d5cdff)"></span>polynomials P₀ … Pₙ (low → high order)</div>
      <div><span class="swatch" style="background:transparent;border:1px dashed #888"></span>center x = a</div>
    `;
  } else if (currentTab === 'B') {
    legend.innerHTML = `
      <div><span class="swatch" style="background:#ffd166"></span>true f(x)</div>
      <div><span class="swatch" style="background:#d5cdff"></span>polynomial Pₙ</div>
      <div><span class="swatch" style="background:#c02a2a;opacity:0.5"></span>actual error f(x) − Pₙ(x)</div>
      <div><span class="swatch" style="background:transparent;border:1px dashed #c02a2a"></span>Lagrange bound (Task 9)</div>
    `;
  } else {
    legend.innerHTML = `<div>(coming soon)</div>`;
  }

  if (currentTab === 'C' || currentTab === 'D') {
    document.getElementById('ovFormula').textContent = `Tab ${currentTab}: coming soon — see design spec`;
    document.getElementById('ovTabReadout').textContent = '';
    return;
  }
  document.getElementById('ovFormula').textContent = p.formula;
  if (currentTab === 'A') {
    const coeffs = coefficientsAt(currentPreset, center, order);
    document.getElementById('ovTabReadout').textContent = formatPolynomialReadout(coeffs, center);
  } else if (currentTab === 'B') {
    document.getElementById('ovTabReadout').textContent =
      `n = ${order}   ·   interval [${intervalLo.toFixed(2)}, ${intervalHi.toFixed(2)}]`;
  }
}
```

- [ ] **Step 4: Visual checkpoint**

Reload, click Tab B. Default state: `sin(x)`, n=5, a=0. Expected:
- Gold sin curve as before
- Single lavender polynomial that hugs sin in the middle and pulls away at the edges
- Translucent red band visible in the regions where polynomial diverges from gold (near ±π and beyond)

Drag order slider 0 → 20: red band visibly shrinks across the window as the polynomial gets better.

Click `eˣ` preset: error band on the negative-x side starts large (Maclaurin of eˣ underestimates), shrinks fast as n grows.

- [ ] **Step 5: Commit**

```bash
git add series.html
git commit -m "$(cat <<'EOF'
Tab B: single polynomial + actual error band

Renders the true curve (gold), one polynomial P_n (lavender), and the
signed error region between them as a translucent red triangle-strip mesh.
Legend swaps per tab. Lagrange envelope and interval handles come next.

EOF
)"
```

---

## Task 9: Tab B — Lagrange remainder envelope

**Files:**
- Modify: `series.html`

Add the dashed red envelope at `Pₙ(x) ± M·|x−a|^(n+1)/(n+1)!`, where `M = max|f^(n+1)(x)|` over the user's interval. For named presets we use closed-form M where it's clean; for `1/(1+x²)` and custom we sample numerically.

- [ ] **Step 1: Add the M-bound calculation helper**

Insert this block in the script, right after the `coefficientsAt` function:

```javascript
// === Lagrange bound calculation ===
// M = max |f^(n+1)(x)| over [lo, hi]. For named presets we have closed forms;
// for everything else, sample 200 points and take the max abs.

function maxDerivativeBound(presetKey, n, lo, hi) {
  // Closed-form bounds for named presets.
  if (presetKey === 'sin' || presetKey === 'cos') return 1;
  if (presetKey === 'exp') {
    // |f^(n+1)| = e^x, monotone increasing → max at hi.
    return Math.exp(hi);
  }
  if (presetKey === 'ln1p') {
    // |f^(n+1)| = n! / (1+x)^(n+1), monotone decreasing in (1+x).
    // Max at the value of x where (1+x) is smallest, i.e. min(|1+lo|, |1+hi|)
    // assuming both are positive (i.e. interval inside (-1, ∞)).
    const minOnePlus = Math.min(1 + lo, 1 + hi);
    if (minOnePlus <= 0) return Infinity;
    return FACTORIAL[n] / Math.pow(minOnePlus, n + 1);
  }
  if (presetKey === 'oneOver1Minus') {
    // |f^(n+1)| = (n+1)! / (1-x)^(n+2), max at x closest to 1.
    const maxX = Math.max(lo, hi);
    if (maxX >= 1) return Infinity;
    return FACTORIAL[n + 1] / Math.pow(1 - maxX, n + 2);
  }
  // Numeric fallback: sample 200 points across [lo, hi].
  const SAMPLES = 200;
  let m = 0;
  for (let i = 0; i <= SAMPLES; i++) {
    const x = lo + (hi - lo) * (i / SAMPLES);
    const v = Math.abs(derivAt(presetKey, x, n + 1));
    if (isFinite(v) && v > m) m = v;
  }
  return m;
}
```

- [ ] **Step 2: Add envelope rendering to `buildTabB`**

In `buildTabB`, after the line `stage.add(buildPolyline(xs, polyClipped, lavender, 1.0));` and before the dashed center line, insert:

```javascript
  // Lagrange envelope: Pn(x) ± M · |x−a|^(n+1) / (n+1)!
  // M is computed over [intervalLo, intervalHi], NOT the full plot window.
  const M = maxDerivativeBound(currentPreset, order, intervalLo, intervalHi);
  if (isFinite(M)) {
    const fact = FACTORIAL[order + 1];
    const envUpYs = xs.map((x, i) => polyYs[i] + M * Math.pow(Math.abs(x - center), order + 1) / fact);
    const envDnYs = xs.map((x, i) => polyYs[i] - M * Math.pow(Math.abs(x - center), order + 1) / fact);
    const envUpClipped = clipY(envUpYs, yMin, yMax);
    const envDnClipped = clipY(envDnYs, yMin, yMax);
    const envColor = new THREE.Color(NEGATIVE_GRADIENT[2]);
    const envUpMat = new THREE.LineDashedMaterial({
      color: envColor, dashSize: 0.12, gapSize: 0.08, transparent: true, opacity: 0.85,
    });
    const envUpGeom = new THREE.BufferGeometry().setFromPoints(
      xs.map((x, i) => new THREE.Vector3(x, envUpClipped[i], 0)).filter(p => isFinite(p.y))
    );
    const envUp = new THREE.Line(envUpGeom, envUpMat);
    envUp.computeLineDistances();
    stage.add(envUp);
    const envDnGeom = new THREE.BufferGeometry().setFromPoints(
      xs.map((x, i) => new THREE.Vector3(x, envDnClipped[i], 0)).filter(p => isFinite(p.y))
    );
    const envDn = new THREE.Line(envDnGeom, envUpMat.clone());
    envDn.computeLineDistances();
    stage.add(envDn);
  }
```

- [ ] **Step 3: Visual checkpoint**

Reload, click Tab B. Expected:
- Two dashed red curves bracketing the lavender polynomial
- Envelope is tightest near x=a (where `|x−a|^(n+1)` is smallest) and widens away from it
- Increasing n: envelope tightens dramatically (factorial in denominator)
- Switching to `eˣ`: envelope is asymmetric (M = e^hi grows fast on the right)
- Switching to `1/(1−x)` with intervalHi near 1: envelope explodes (M → ∞ near the singularity)

The actual error band (red fill) should always lie inside the dashed envelope. If it doesn't, the bound is wrong — debug `maxDerivativeBound` for that preset.

- [ ] **Step 4: Commit**

```bash
git add series.html
git commit -m "$(cat <<'EOF'
Tab B: Lagrange remainder envelope as dashed red curves

M = max|f^(n+1)| over the user-defined interval, closed-form for the named
presets where it's clean (sin/cos = 1; exp grows; ln1p and 1/(1-x) blow up
near singularities), numeric fallback otherwise. Envelope drawn as two
dashed lines bracketing the polynomial. Actual error band stays inside.

EOF
)"
```

---

## Task 10: Tab B — interval handles (draggable)

**Files:**
- Modify: `series.html`

Currently `intervalLo` and `intervalHi` are wired to two range sliders in the footer. This task adds visible vertical lines at those positions in the scene and recomputes M when either slider moves.

- [ ] **Step 1: Wire the interval sliders to state and rebuild**

Insert this block in the script after the `centerSlider` event listener:

```javascript
function updateIntervalLabel() {
  document.getElementById('lblInterval').textContent =
    `[${intervalLo.toFixed(2)}, ${intervalHi.toFixed(2)}]`;
}

document.getElementById('intervalLoSlider').addEventListener('input', e => {
  intervalLo = parseFloat(e.target.value);
  if (intervalLo > center - 0.05) intervalLo = center - 0.05;  // keep below center
  updateIntervalLabel();
  rebuild();
});

document.getElementById('intervalHiSlider').addEventListener('input', e => {
  intervalHi = parseFloat(e.target.value);
  if (intervalHi < center + 0.05) intervalHi = center + 0.05;  // keep above center
  updateIntervalLabel();
  rebuild();
});
```

- [ ] **Step 2: Draw interval handles as vertical lines in `buildTabB`**

In `buildTabB`, right after the dashed center line is added (and before the anchor dot), insert:

```javascript
  // Interval handles: two solid red verticals at intervalLo and intervalHi.
  const handleColor = NEGATIVE_GRADIENT[1];
  for (const xPos of [intervalLo, intervalHi]) {
    const geom = new THREE.BufferGeometry().setFromPoints([
      new THREE.Vector3(xPos, yMin - 0.3, 0),
      new THREE.Vector3(xPos, yMax + 0.3, 0),
    ]);
    const mat = new THREE.LineBasicMaterial({
      color: handleColor, transparent: true, opacity: 0.5,
    });
    stage.add(new THREE.Line(geom, mat));
  }
```

- [ ] **Step 3: Visual checkpoint**

Reload, click Tab B. Two faint red vertical lines should mark `[−1.5, 1.5]` by default. Drag the interval-low slider rightward (closer to 0): the left handle moves in; the dashed Lagrange envelope tightens (M shrinks because we're sampling f^(n+1) on a smaller window).

Try `1/(1−x)`: drag interval-hi from 1.5 toward 0.5. The dashed envelope should shrink dramatically as the singularity at x=1 leaves the interval.

- [ ] **Step 4: Commit**

```bash
git add series.html
git commit -m "$(cat <<'EOF'
Tab B: draggable interval handles for the M-bound

Two red vertical lines mark [intervalLo, intervalHi]. Sliders in the footer
control them; M is recomputed on every drag and the dashed Lagrange
envelope re-tightens. Handles are clamped to stay on opposite sides of the
center point a.

EOF
)"
```

---

## Task 11: Tab B — readout panel (M, bound, actual error)

**Files:**
- Modify: `series.html`

Replace the placeholder `n = X · interval [...]` readout with the full educational payload: M, the bound at the interval edge, the actual maximum error.

- [ ] **Step 1: Add a helper that returns the actual max error over the interval**

Insert in the script right after `maxDerivativeBound`:

```javascript
// Actual max |f(x) − P_n(x)| over [lo, hi]. Sample 200 points.
function actualMaxError(presetKey, coeffs, a, lo, hi) {
  const f = PRESETS[presetKey].f;
  let m = 0;
  for (let i = 0; i <= 200; i++) {
    const x = lo + (hi - lo) * (i / 200);
    const err = Math.abs(f(x) - taylorPoly(coeffs, a, x));
    if (isFinite(err) && err > m) m = err;
  }
  return m;
}
```

- [ ] **Step 2: Update `syncOverlay`'s Tab B branch with the rich readout**

In `syncOverlay`, find:
```javascript
  } else if (currentTab === 'B') {
    document.getElementById('ovTabReadout').textContent =
      `n = ${order}   ·   interval [${intervalLo.toFixed(2)}, ${intervalHi.toFixed(2)}]`;
  }
```
Replace with:
```javascript
  } else if (currentTab === 'B') {
    const coeffs = coefficientsAt(currentPreset, center, order);
    const M = maxDerivativeBound(currentPreset, order, intervalLo, intervalHi);
    const fact = FACTORIAL[order + 1];
    // Worst-case |x − a|^(n+1) at the interval edge furthest from center.
    const dEdge = Math.max(Math.abs(intervalHi - center), Math.abs(intervalLo - center));
    const bound = isFinite(M) ? M * Math.pow(dEdge, order + 1) / fact : Infinity;
    const actual = actualMaxError(currentPreset, coeffs, center, intervalLo, intervalHi);
    const fmtM = isFinite(M) ? M.toExponential(2) : '∞';
    const fmtBound = isFinite(bound) ? bound.toExponential(2) : '∞';
    const fmtActual = actual.toExponential(2);
    document.getElementById('ovTabReadout').innerHTML =
      `n = <b>${order}</b>   ·   interval [${intervalLo.toFixed(2)}, ${intervalHi.toFixed(2)}]<br>` +
      `M = max|f<sup>(${order + 1})</sup>| ≈ <b>${fmtM}</b><br>` +
      `Lagrange bound at edge: <b>${fmtBound}</b><br>` +
      `Actual max error: <b>${fmtActual}</b>`;
  }
```

- [ ] **Step 3: Visual checkpoint**

Reload, Tab B, default state. Overlay should now show:
```
y = sin(x)
Maclaurin: x − x³/3! + x⁵/5! − …
n = 5   ·   interval [−1.50, 1.50]
M = max|f^(6)| ≈ 1.00e+0
Lagrange bound at edge: 1.58e-2
Actual max error: 1.41e-3
```

The "Actual max error" should always be ≤ "Lagrange bound at edge" — that's the whole point of the bound. If actual ever exceeds bound, debug.

Drag order to 10: bound should drop sharply (factorial), actual should drop too. Drag interval-hi from 1.5 toward 0.5: both numbers shrink.

- [ ] **Step 4: Commit**

```bash
git add series.html
git commit -m "$(cat <<'EOF'
Tab B: rich readout — M, Lagrange bound, actual max error

Overlay now shows the full educational payload: max|f^(n+1)| over the
interval, the Lagrange bound at the interval edge, and the actual maximum
error sampled over 200 points. The "actual ≤ bound" comparison is the
educational moment. Numbers in scientific notation; ∞ shown when the bound
includes a singularity.

EOF
)"
```

---

## Task 12: Landing page — third method card

**Files:**
- Modify: `index.html`

Add a third card for "Taylor Series" linking to `series.html`. Update the title from "Volumes of Revolution" to a phrase that fits all three. Update the CSS grid to handle three cards.

- [ ] **Step 1: Update title and CSS grid**

In `index.html`, find:
```html
<title>Volumes of Revolution · 3D Explorer</title>
```
Replace with:
```html
<title>Calculus · 3D Explorer</title>
```

Find:
```html
<h1>Volumes of <span class="accent">Revolution.</span></h1>
```
Replace with:
```html
<h1>Calculus, <span class="accent">visualized.</span></h1>
```

Find the CSS block:
```css
@media (min-width: 880px) {
  .methods { grid-template-columns: 1fr 1fr; }
}
```
Replace with:
```css
@media (min-width: 880px) {
  .methods { grid-template-columns: 1fr 1fr; }
}
@media (min-width: 1180px) {
  .methods { grid-template-columns: repeat(3, 1fr); }
}
```

- [ ] **Step 2: Add the third card after the washer card**

Find the closing `</a>` of the washer card, then the closing `</div>` of `.methods`. Insert this card before the closing `</div>`:

```html
    <a class="method-card" href="series.html" aria-label="Open Taylor Series explorer">
      <div class="frame">
        <img src="series-preview.png" alt="Polynomial approximations converging on a sine curve" width="1280" height="800">
      </div>
      <div class="meta">
        <h2>Taylor <b>series.</b></h2>
        <div class="formula">f(x) ≈ Σ fⁿ(a)/n! · (x−a)ⁿ</div>
        <div class="open">
          <span>Open</span>
          <span class="arrow"></span>
        </div>
      </div>
    </a>
```

- [ ] **Step 3: Visual checkpoint**

Reload `http://localhost:8765/index.html`. Expected:
- Title now reads "Calculus, *visualized.*"
- Three cards visible at desktop (≥1180px), two cards on mid-width, one card on mobile
- Third card shows a broken image placeholder for `series-preview.png` (we capture the screenshot in Task 13) — this is expected for now
- Click the Taylor card → navigates to `series.html`

The broken image is OK at this step. Don't commit until Task 13 captures the screenshot.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Landing page: third card for Taylor Series

Broaden the title from "Volumes of Revolution" to "Calculus, visualized."
Add a 3-column breakpoint at 1180px. New card links to series.html. The
preview image is captured in the next commit.

EOF
)"
```

---

## Task 13: Capture series-preview.png

**Files:**
- Create: `series-preview.png`

This is a manual screenshot step. The other previews (`shell-preview.png`, `washer-preview.png`) are 1280×800 PNGs of the live page. Match that.

- [ ] **Step 1: Set up the canonical preview state**

Open `http://localhost:8765/series.html` in a real browser (Chrome/Firefox, not the preview tool — see "Local dev" note about WebGL hangs). Set:
- Function: `y = sin(x)` (default)
- Tab: A (default)
- Order n: **8** (drag the slider; default 5 is OK but 8 reads more impressive in a thumbnail)
- Center a: 0 (default)

Wait for the polynomial stack to render fully. The composition should show the gold sin wave with 9 polynomials fanning outward — visually rich.

- [ ] **Step 2: Capture a 1280×800 screenshot**

The simplest way: resize the browser window so the scene-wrap area is roughly 1280×800, then use the OS screenshot tool (Windows Snip & Sketch with Win+Shift+S). Crop to just the scene canvas + a sliver of the overlay so it visually matches `washer-preview.png` and `shell-preview.png`.

If precise sizing matters, use Chrome DevTools' device toolbar (Ctrl+Shift+M), set a custom 1280×800 viewport, then DevTools → three-dot menu → "Capture full size screenshot."

Save the result as `series-preview.png` in the project root (next to `shell-preview.png`).

- [ ] **Step 3: Verify the image**

Reload the landing page. The third card should now show the screenshot instead of the broken-image icon.

- [ ] **Step 4: Commit**

```bash
git add series-preview.png
git commit -m "$(cat <<'EOF'
Add series-preview.png for the landing card

1280x800 screenshot of Tab A with sin(x) Maclaurin at n=8 — the polynomial
stack fanning out from origin reads cleanly at thumbnail size.

EOF
)"
```

---

## Task 14: Update CLAUDE.md with series page architecture notes

**Files:**
- Modify: `CLAUDE.md`

Add a section to the project's CLAUDE.md so the next session can pick up the series page without re-reading the spec.

- [ ] **Step 1: Update the file-tree section**

In `CLAUDE.md`, find the project-structure block:
```
calc-volumes-3d/
├── index.html        # the entire app (UI + Three.js scene + math)
├── README.md         # public-facing project description
├── CLAUDE.md         # this file
├── .gitignore        # ignores .claude/ and OS junk
└── .claude/
    └── launch.json   # local preview config (gitignored)
```

Replace with:
```
calc-volumes-3d/
├── index.html        # landing page (links to the three method explorers)
├── shell.html        # shell method 3D explorer (the original)
├── washer.html       # washer method 3D explorer
├── series.html       # Taylor series 2D explorer (4 tabs; A & B built, C & D spec'd)
├── shell-preview.png
├── washer-preview.png
├── series-preview.png
├── README.md
├── CLAUDE.md
├── docs/superpowers/
│   ├── specs/        # design docs (one per major feature)
│   └── plans/        # implementation plans tied to specs
├── .gitignore
└── .claude/
    └── launch.json
```

- [ ] **Step 2: Add a "Series page architecture" section**

Insert this block in `CLAUDE.md` between the "Architecture (mental model)" section (which describes shell.html) and the "Conventions" section:

```markdown
## Series page architecture (series.html)

A 2D Taylor/Maclaurin explorer with four tabs sharing one chassis:

1. **Same Three.js + importmap pattern** as shell/washer, but with
   `OrthographicCamera` looking down the +z axis so the xy plane reads as
   2D. Same disposal/rebuild pattern. Same color tokens. `OrbitControls`
   with `enableRotate = false` — pan and zoom only.
2. **Function library is different:** sin, cos, exp, ln(1+x), 1/(1-x),
   1/(1+x²), and custom. Polynomial families like √x or x²/2 are
   pedagogically boring here (their Taylor series is themselves).
3. **Derivative core (`derivAt`)** uses closed-form derivative formulas for
   the named presets (clean, no numeric noise). Custom expressions and
   1/(1+x²) fall back to recursive 5-point finite differences. Cap order
   at 20 — finite diff degrades past that.
4. **Coefficient cache (`coefficientsAt`)** memoizes `f^(k)(a)/k!` arrays
   keyed by `(presetKey, a, n)`. Polynomial evaluation across 400 plot
   points reuses the same coefficients. Cleared on custom-apply. LRU-
   bounded at 500 entries.
5. **Tab bar dispatches to four scene builders** (`buildTabA`, `buildTabB`,
   etc.) that each add geometry to the shared `stage` group. `rebuild()`
   disposes the stage and calls the right builder. Same pattern as the
   shell explorer's `rebuild`.
6. **Y-axis clipping in `clipY`** is critical — high-order polynomials shoot
   to ±∞ outside the convergence radius and would crush the rest of the
   scene. Clip everything to slightly outside the visible y-range so the
   polynomial visibly slams into the viewport edge.

**Tab status:**
- **A (polynomial stack):** built. Order slider, play, formula readout.
- **B (error bounds):** built. Single P_n + actual error band + dashed
  Lagrange envelope + draggable interval handles + M/bound/actual readout.
- **C (convergence radius):** spec'd in
  `docs/superpowers/specs/2026-05-02-taylor-series-explorer-design.md`,
  not built. Tab body is empty; overlay shows "coming soon".
- **D (center point):** spec'd in same doc, not built.

**Inline self-tests:** the derivative core and the polynomial evaluator
each have an IIFE test block that runs at page load and logs PASS/FAIL to
console. No test framework — these are `console.assert`-style. If you
modify either module, reload and check the console.
```

- [ ] **Step 3: Update the "Note to next Claude session" block**

In `CLAUDE.md`, find the "Note to next Claude session" section near the end. Replace its contents with:

```markdown
## Note to next Claude session

(Picking this back up — read this section before doing anything substantive.)

**Status as of last edit:** Three explorer pages live (shell, washer, series).
The series page has tabs A and B fully built; tabs C and D are spec'd in
`docs/superpowers/specs/2026-05-02-taylor-series-explorer-design.md` but
not implemented — their tab bodies render nothing and the overlay shows
"coming soon". Most recent push: see `git log -1`.

**Pending:** still need to send GitHub usernames so the collaborator
invites can fire (`gh api -X PUT /repos/jweivy/multi-final-project/collaborators/USERNAME -f permission=push`).

**Likely next requests:**
1. **Build Tabs C and D of the series page.** Spec is complete; pick up
   from the build-order section of the spec doc.
2. **Annotate shells/washers** with their individual contribution to the
   integral (e.g. floating "+0.23" labels per shell).
3. **Toggle for absolute vs signed volume** on the shell/washer pages.
4. **2D companion plot** on the shell page showing the curve in xz with
   the active shell's strip highlighted.
5. **x-axis revolution toggle** for shells (more invasive — changes the
   coordinate setup).

**Watch for:** any change to series.html that touches `derivAt` or
`coefficientsAt` should be tested by reloading and checking the console
for the two PASS lines from the inline self-tests. Any change to the
polynomial stack rendering should be tested with `1/(1+x²)` because
that's the function that exercises the y-clipping at the convergence
radius edge.

**Don't suggest splitting any of the .html files.** Same monolithic
constraint as before — each page should stay under ~1500 lines or it's
time for a conversation, not a refactor.
```

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md
git commit -m "$(cat <<'EOF'
CLAUDE.md: document series page architecture

Add file-tree entries for series.html and the docs/ folder. New section
describes the chassis, derivative core, coefficient cache, tab dispatch,
and y-clipping. Updates the next-session note with current status and
likely follow-up requests (most notably, building Tabs C and D from the
existing spec).

EOF
)"
```

---

## Self-review against spec

Working through the spec's requirements:

- **Architecture / single file / Three.js OrthographicCamera** — Task 1 sets this up.
- **Function library (sin, cos, exp, ln1p, 1/(1-x), 1/(1+x²), custom)** — Task 1 step 3.
- **Center / order / play / reset shared controls** — Task 1 step 7 + Task 5.
- **Tab bar A/B/C/D** — Task 1 step 4.
- **Numeric derivative core with closed-form preset table** — Task 2.
- **Polynomial evaluator with caching** — Task 3.
- **Tab A: polynomial stack, anchor dot, dashed center, formula readout, play, custom expression, y-clipping** — Tasks 4–7.
- **Tab B: single polynomial, error band, Lagrange envelope, interval handles, M/bound/actual readout** — Tasks 8–11.
- **Tab C / D: scaffolded as "coming soon"** — Task 1 step 4 (tab buttons exist) + the `placeholderTab` stub in step 8 (renders nothing). Verified end-to-end after Task 11.
- **Landing page third card** — Task 12.
- **series-preview.png** — Task 13.
- **CLAUDE.md update** — Task 14.

No gaps. No placeholders in the plan steps. Type/method names checked: `derivAt`, `coefficientsAt`, `taylorPoly`, `buildPolyline`, `buildBand`, `clipY`, `sampleX`, `fitCameraY`, `maxDerivativeBound`, `actualMaxError`, `formatPolynomialReadout`, `gradientColor`, `disposeGroup`, `compileExpression`, `placeholderTab`, `buildTabA`, `buildTabB`, `syncOverlay`, `rebuild`, `stopPlay`, `updateIntervalLabel` — all defined where first used.

Risk noted from spec — Three.js `LineDashedMaterial` rendering can be flaky. Task 9 uses it for the Lagrange envelope and Tab A uses it for the center line; if either looks bad in practice, swap to short tube segments with gaps. Both center line and envelope call `.computeLineDistances()` as required by the dashed material.
