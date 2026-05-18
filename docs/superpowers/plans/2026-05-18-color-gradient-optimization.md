# Color Gradient Optimization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the washed-out 3-stop gradients in shell.html and washer.html with richer 5-stop arrays and a generalized N-stop interpolation function.

**Architecture:** Three symbol changes per file — `POSITIVE_GRADIENT`, `NEGATIVE_GRADIENT`, and `gradientColor` — applied identically to `shell.html` and `washer.html`. No math, controls, or layout logic is touched. Legend swatches update automatically since they read from the gradient constants at render time.

**Tech Stack:** Vanilla JS, Three.js r160 (already loaded via importmap). No new dependencies.

**Spec:** `docs/superpowers/specs/2026-05-17-color-gradient-optimization-design.md`

---

## Files

| File | Change |
|------|--------|
| `shell.html` | Update `POSITIVE_GRADIENT`, `NEGATIVE_GRADIENT`, rewrite `gradientColor` |
| `washer.html` | Same three changes, identical values |

---

### Task 1: Create branch

- [ ] **Step 1: Confirm we're on main and create the branch**

```bash
git checkout main
git pull
git checkout -b color-optimization
```

Expected: `Switched to a new branch 'color-optimization'`

---

### Task 2: Update `shell.html`

**Files:**
- Modify: `shell.html` (lines ~597–605 for constants, ~748–759 for gradientColor)

- [ ] **Step 1: Replace `POSITIVE_GRADIENT` constant**

Find this exact block (around line 600):
```js
const POSITIVE_GRADIENT = [0x10204a, 0x6750d8, 0xd5cdff];
```

Replace with:
```js
const POSITIVE_GRADIENT = [0x1e1060, 0x3d28aa, 0x6750d8, 0x9270e0, 0xa888ee];
```

- [ ] **Step 2: Replace `NEGATIVE_GRADIENT` constant**

Find:
```js
const NEGATIVE_GRADIENT = [0x4a1010, 0xc02a2a, 0xff7070];
```

Replace with:
```js
const NEGATIVE_GRADIENT = [0x3a0a0a, 0x6e1515, 0xa82020, 0xc43535, 0xd04848];
```

- [ ] **Step 3: Rewrite `gradientColor` to handle N stops**

Find this exact function (around line 748):
```js
// Sample a 3-stop gradient at parameter t ∈ [0, 1].
function gradientColor(stops, t) {
  t = Math.max(0, Math.min(1, t));
  const [c0, c1, c2] = stops.map(h => new THREE.Color(h));
  if (t <= 0.5) {
    const u = t / 0.5;
    return new THREE.Color().lerpColors(c0, c1, u);
  } else {
    const u = (t - 0.5) / 0.5;
    return new THREE.Color().lerpColors(c1, c2, u);
  }
}
```

Replace with:
```js
// Sample an N-stop gradient at parameter t ∈ [0, 1].
function gradientColor(stops, t) {
  t = Math.max(0, Math.min(1, t));
  const colors = stops.map(h => new THREE.Color(h));
  const n = colors.length - 1;
  const seg = Math.min(Math.floor(t * n), n - 1);
  const u = (t * n) - seg;
  return new THREE.Color().lerpColors(colors[seg], colors[seg + 1], u);
}
```

- [ ] **Step 4: Commit shell.html**

```bash
git add shell.html
git commit -m "feat(shell): richer 5-stop gradient, generalize gradientColor to N stops

- POSITIVE_GRADIENT: deep indigo → medium purple (no more washed-out lavender)
- NEGATIVE_GRADIENT: near-black crimson → rich red (no more salmon fade)
- gradientColor: generalized from hardcoded 3-stop to N-stop piecewise lerp"
```

---

### Task 3: Update `washer.html`

**Files:**
- Modify: `washer.html` (same line ranges as shell.html — identical constants and function)

- [ ] **Step 1: Replace `POSITIVE_GRADIENT` constant**

Find:
```js
const POSITIVE_GRADIENT = [0x10204a, 0x6750d8, 0xd5cdff];
```

Replace with:
```js
const POSITIVE_GRADIENT = [0x1e1060, 0x3d28aa, 0x6750d8, 0x9270e0, 0xa888ee];
```

- [ ] **Step 2: Replace `NEGATIVE_GRADIENT` constant**

Find:
```js
const NEGATIVE_GRADIENT = [0x4a1010, 0xc02a2a, 0xff7070];
```

Replace with:
```js
const NEGATIVE_GRADIENT = [0x3a0a0a, 0x6e1515, 0xa82020, 0xc43535, 0xd04848];
```

- [ ] **Step 3: Rewrite `gradientColor` to handle N stops**

Find:
```js
// Sample a 3-stop gradient at parameter t ∈ [0, 1].
function gradientColor(stops, t) {
  t = Math.max(0, Math.min(1, t));
  const [c0, c1, c2] = stops.map(h => new THREE.Color(h));
  if (t <= 0.5) {
    const u = t / 0.5;
    return new THREE.Color().lerpColors(c0, c1, u);
  } else {
    const u = (t - 0.5) / 0.5;
    return new THREE.Color().lerpColors(c1, c2, u);
  }
}
```

Replace with:
```js
// Sample an N-stop gradient at parameter t ∈ [0, 1].
function gradientColor(stops, t) {
  t = Math.max(0, Math.min(1, t));
  const colors = stops.map(h => new THREE.Color(h));
  const n = colors.length - 1;
  const seg = Math.min(Math.floor(t * n), n - 1);
  const u = (t * n) - seg;
  return new THREE.Color().lerpColors(colors[seg], colors[seg + 1], u);
}
```

- [ ] **Step 4: Commit washer.html**

```bash
git add washer.html
git commit -m "feat(washer): richer 5-stop gradient, generalize gradientColor to N stops

Mirror of shell.html changes — identical constants and gradientColor rewrite."
```

---

### Task 4: Verify in browser

Start the preview server if it isn't already running:
```bash
py -3.13 -m http.server 8765 --directory .
```
Open `http://localhost:8765`.

- [ ] **shell.html — positive shells**
  - Open `shell.html`. Cycle through all presets (√x, x²/2, sin(x)+0.2).
  - At shell density **4**, confirm each shell is a clearly distinct, rich violet shade — no salmon or lavender.
  - At shell density **40**, confirm a smooth unbroken gradient from deep indigo to medium purple.
  - Legend swatch (top-left overlay) reflects the new gradient.

- [ ] **shell.html — negative shells**
  - Set custom preset to `sin(x)` on `[0, 2π]`. Increase density to ~20.
  - Confirm shells below the axis are deep crimson → rich red — no salmon/pink.

- [ ] **washer.html — positive washers**
  - Open `washer.html`. Cycle through all washer presets.
  - Same density checks as above (4 and 40). Confirm smooth rich gradient.

- [ ] **No console errors** — open DevTools, confirm zero errors on both pages.

- [ ] **Final commit if any last-minute tweaks were made**, then push:

```bash
git push -u origin color-optimization
```
