# Color Gradient Optimization — Design Spec

**Date:** 2026-05-17  
**Branch:** `color-optimization`  
**Files affected:** `shell.html`, `washer.html`

---

## Problem

The current shell/washer gradients have three issues:

1. **Washes out at the high end** — positive gradient ends at near-white lavender (`#d5cdff`); negative ends at salmon (`#ff7070`). Both look faded against the dark background.
2. **Chunky at low shell counts** — with only 3 color stops and piecewise-linear RGB interpolation, the visual jump between adjacent shells is large when density is low (4–8 shells).
3. **Palette not bold enough** — the overall range feels weak; the solid reads as muted rather than rich.

---

## Solution

**Option B: new colors + generalize to N-stop interpolation.**

Replace both gradient constants with 5-stop arrays that stay saturated throughout. Rewrite `gradientColor` to handle any number of stops uniformly (one segment per gap between stops).

---

## New Color Constants

### `POSITIVE_GRADIENT` (shells/washers where f(x) ≥ 0)

```js
const POSITIVE_GRADIENT = [0x1e1060, 0x3d28aa, 0x6750d8, 0x9270e0, 0xa888ee];
```

| Stop | Hex | Description |
|------|-----|-------------|
| 0 | `#1e1060` | Deep indigo — innermost / first shell |
| 1 | `#3d28aa` | Dark violet |
| 2 | `#6750d8` | Mid violet (same anchor as before) |
| 3 | `#9270e0` | Soft violet |
| 4 | `#a888ee` | Medium purple — outermost / last shell |

### `NEGATIVE_GRADIENT` (shells where f(x) < 0)

```js
const NEGATIVE_GRADIENT = [0x3a0a0a, 0x6e1515, 0xa82020, 0xc43535, 0xd04848];
```

| Stop | Hex | Description |
|------|-----|-------------|
| 0 | `#3a0a0a` | Near-black crimson — innermost |
| 1 | `#6e1515` | Deep red |
| 2 | `#a82020` | Mid red |
| 3 | `#c43535` | Medium-bright red |
| 4 | `#d04848` | Rich red — outermost (no salmon fade) |

---

## Code Change: `gradientColor`

**Current (hardcoded 3-stop):**
```js
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

**New (generalized N-stop):**
```js
function gradientColor(stops, t) {
  t = Math.max(0, Math.min(1, t));
  const colors = stops.map(h => new THREE.Color(h));
  const n = colors.length - 1;
  const seg = Math.min(Math.floor(t * n), n - 1);
  const u = (t * n) - seg;
  return new THREE.Color().lerpColors(colors[seg], colors[seg + 1], u);
}
```

This change is identical in both `shell.html` and `washer.html`.

---

## Scope

- `shell.html`: update `POSITIVE_GRADIENT`, `NEGATIVE_GRADIENT`, `gradientColor`
- `washer.html`: same three changes, identical values
- `series.html`: no changes
- `index.html`: no changes
- Legend swatches update automatically (they already read from the gradient constants at render time)
- No changes to any math, controls, camera, or layout logic

---

## Out of Scope

- Perceptual (OKLab) interpolation — decided against; effect too subtle for the added complexity
- Per-preset gradient overrides — deliberately removed in a prior session; not reintroduced here
- Any changes to the `series.html` color scheme

---

## Testing

After implementation, verify in browser at `http://localhost:8765`:

1. `shell.html` — cycle through all presets at shell density 4, 12, 40. Confirm gradient looks rich and unbroken at all densities.
2. `shell.html` — set custom preset to `sin(x)` on `[0, 2π]`. Confirm positive shells (blue-purple) and negative shells (red) both look correct and neither fades to near-white/salmon.
3. `washer.html` — repeat step 1 with all washer presets.
4. Legend swatch on both pages reflects the new gradient.
5. No console errors.
