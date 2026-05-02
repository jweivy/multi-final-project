# Taylor Series Explorer — Design

**Date:** 2026-05-02
**Status:** Approved. Tabs A and B to be implemented; tabs C and D spec'd but deferred.
**Sibling pages:** `shell.html`, `washer.html`. Landing page: `index.html`.

## Goal

Build a fourth page in the calc-volumes-3d project — `series.html` — that visualizes Taylor and Maclaurin series alongside their error bounds and convergence behavior. Add a third card to the landing page linking to it.

The page is organized as a single `series.html` with an internal tab bar exposing four lessons, each centered on one "aha" insight:

- **Tab A — "Polynomials sneak up."** Watch P₀ → Pₙ stack toward the true curve as order grows.
- **Tab B — "The error has a shape."** Visualize the actual error as a red band and the Lagrange remainder as a dashed envelope; squeeze both with `n` and the interval.
- **Tab C — "The radius of convergence is real."** Color the convergence band blue and the divergence zones red; watch high-order polynomials explode outside `[a−R, a+R]`.
- **Tab D — "Maclaurin is Taylor at a=0."** Make the center `a` the star — drag it along the curve, snap back to 0 to call out Maclaurin.

Tabs C and D are fully designed below but not implemented in this pass. Their bodies in the live page show a "Coming soon" placeholder.

## Non-goals

- Symbolic differentiation. Numeric derivatives + hardcoded derivative formulas for named presets only.
- A bundler, npm, or any framework. Continue the project's "open the file, it works" pattern.
- Splitting `series.html` into multiple files. Keep monolithic, like the other pages.
- TeX-style math input (`\sin`, `\frac{...}`) — same constraint as `shell.html`.
- 3D rendering. Taylor series is 2D; we'll use Three.js with an `OrthographicCamera` for codebase consistency, not for actual 3D.

## Architecture

### Files

```
calc-volumes-3d/
├── index.html         # add a third method card linking to series.html
├── shell.html         # unchanged
├── washer.html        # unchanged
├── series.html        # NEW — this design
└── CLAUDE.md          # update with series-page architecture notes
```

### Shared chassis (used by all four tabs)

- **Rendering:** Three.js r160 from CDN via the existing `importmap` pattern. `OrthographicCamera` looking straight down at the xy plane. Same disposal/rebuild pattern as `shell.html` (single `stage` group, dispose-and-rebuild on every state change).
- **Color system:** reuse `POSITIVE_GRADIENT` (blue → indigo → lavender) for polynomial-order ramping (P₀ darkest, Pₙ palest). Reuse `NEGATIVE_GRADIENT` (deep red → bright red) for the error band (Tab B) and divergence zones (Tab C). Gold for the true `f(x)` curve, matching the shell-method curve color.
- **Typography & layout:** match the editorial style established in shell/washer. Big serif heading at top, sans-serif body and overlays. Same color tokens.

### Function library

Different from shell/washer because polynomial families are boring for Taylor (their series is themselves) and `√x`/`x²/2` don't teach much:

- `sin(x)` — default for tabs A, B, D
- `cos(x)`
- `eˣ`
- `ln(1+x)`
- `1/(1−x)`
- `1/(1+x²)` — default for Tab C (the dramatic convergence-radius example)
- `✏️ custom` — same expression parser as `shell.html`'s `compileExpression`

Same preset-pill row pattern as the other pages, with the same custom-equation form (visible only when "custom" is active).

### Shared controls

- **Center point `a`** — slider. Default 0 (so default is Maclaurin). Tabs A, B, C lock this slider; Tab D unlocks it and makes it the star.
- **Order `n`** — slider 0–20.
- **▶ play button + speed dropdown** — same pattern and speeds as shell.html (slow / normal / fast / blitz).
- **↺ reset view** — recenters the orthographic camera.

### Tab bar

Four pills at the top of the page: `A · B · C · D`, with the long titles below for clarity ("Building up", "Error bounds", "Convergence", "Center point"). Tab state is stored in a `currentTab` variable; `rebuild()` switches on it. Each tab body is a function that adds geometry to the shared `stage` group.

### Numeric derivative core

A central module `derivatives(f, x, k)` returning the k-th derivative of `f` at `x`. Used everywhere a Taylor coefficient is needed.

- **For named presets:** hardcoded closed-form derivative formulas. Clean, fast, no numeric noise.
  - `sin(x)` → cycle of `sin, cos, -sin, -cos` per `k mod 4`.
  - `cos(x)` → cycle of `cos, -sin, -cos, sin` per `k mod 4`.
  - `eˣ` → `eˣ` for all `k`.
  - `ln(1+x)` → `(-1)^(k-1) · (k-1)! / (1+x)^k` for `k ≥ 1`; `ln(1+x)` for `k=0`.
  - `1/(1−x)` → `k! / (1−x)^(k+1)`.
  - `1/(1+x²)` → no closed form that's clean; use the recursion `(1+x²)·yⁿ⁺¹ + 2(n+1)x·yⁿ + n(n+1)·yⁿ⁻¹ = 0` to derive coefficients of the Maclaurin expansion (or fall back to numeric for off-zero centers).
- **For custom expressions:** recursive 5-point central finite difference. Cap order at n=20; degrades past that. Step size `h = 1e-3` (tuned in implementation).
- **Caching:** for the named presets, the per-preset derivative tables are pure functions of `(x, k)` so no cache is needed. For custom expressions, cache the Taylor coefficients at the current `(a, n)` so polynomial evaluation across 400 plot points is O(n) per point, not O(n) recomputes.

### Polynomial evaluator

`taylorPoly(coeffs, a, x)` returns `Σ coeffs[k] · (x − a)^k`. Horner's method for stability: `((coeffs[n]·(x−a) + coeffs[n−1])·(x−a) + …) + coeffs[0]`.

`coeffs[k] = derivatives(f, a, k) / k!`. Compute the coefficient array once per `(a, n)` change, reuse across the 400 sample points.

## Tabs

### Tab A — Polynomials sneak up

**Visual:**
- Gold curve = true `f(x)` over the function-appropriate window (e.g. `[−2π, 2π]` for trig, `[−2, 2]` for `1/(1+x²)`).
- Stack of polynomial curves P₀, P₁, …, Pₙ. Color of P_k = `POSITIVE_GRADIENT` sampled at `k/n`. P₀ darkest blue, Pₙ palest lavender.
- Vertical dashed line at `x = a`.
- Anchor dot at `(a, f(a))`.

**Interaction:**
- Order slider `n` (0–20). All P_k for `k ≤ n` are drawn.
- ▶ play animates `n` from 0 to 20.
- Live formula readout in overlay. Renders the symbolic form with numeric coefficients filled in:
  ```
  P₅(x) = sin(0) + cos(0)·x − sin(0)/2!·x² − cos(0)/3!·x³ + …
        = 0 + 1·x + 0 − 0.1667·x³ + 0 + 0.00833·x⁵
  ```
  Updates on `n` and `a` change.

**Math:**
- Compute `coeffs[0..n]` once per `(a, n)` change.
- For each P_k, sample 400 points across the visible window, evaluate `taylorPoly(coeffs.slice(0, k+1), a, x)`, build a `TubeGeometry` with the gold-curve trick from `shell.html`.

**Y-axis clipping:** clamp visible y to `[min(f) − 2, max(f) + 2]` over the visible window. High-order polynomials can shoot off; let them, but clip the rendered tube so they don't crush the rest of the scene.

### Tab B — The error has a shape

**Visual:**
- Gold curve = true `f(x)`.
- Single polynomial Pₙ in pale lavender (top of `POSITIVE_GRADIENT`).
- **Red error band** filled between Pₙ and `f(x)` — the actual signed error `f(x) − Pₙ(x)`. Translucent red, sampled from `NEGATIVE_GRADIENT`.
- **Lagrange envelope** drawn as two dashed red lines at `Pₙ(x) ± M · |x − a|^(n+1) / (n+1)!`, where `M = max |f^(n+1)(x)|` over the user-controlled interval `[x_lo, x_hi]`.
- Center anchor dot at `(a, f(a))` and dashed vertical line at `x = a`.

**Interaction:**
- Order slider `n` (0–20). As `n` grows: Pₙ tightens onto `f`, and the dashed envelope pinches in dramatically (factorial growth).
- **Two draggable interval handles** at `x_lo` and `x_hi` — vertical lines that snap to the nearest sample point on drag. Recomputes `M` on every drag end. The visualization point: smaller interval → smaller M → tighter bound.
- Live readout panel in overlay:
  ```
  n = 5
  Interval: [−1.5, 1.5]
  M = max|f⁽⁶⁾| ≈ 1.00
  Bound at edge: |R₅| ≤ M · |x − a|⁶ / 6! ≈ 0.0158
  Actual max error ≈ 0.0009
  ```
  The "actual ≤ bound" comparison is the educational moment.

**Math:**
- Error band: triangle-strip mesh between two polylines (Pₙ and `f`), using a custom `BufferGeometry`. Red translucent material with `transparent: true, opacity: 0.35`.
- Lagrange envelope: two `TubeGeometry` strips (or `LineDashedMaterial` lines) at `Pₙ ± M·|x−a|^(n+1)/(n+1)!`. Sample at 400 points.
- `M` computation: for named presets, hardcoded:
  - `sin`, `cos` → `M = 1` always
  - `eˣ` over `[x_lo, x_hi]` → `M = e^x_hi`
  - `ln(1+x)` → `M = n! / (1 + min(|x_lo|, |x_hi|))^(n+1)` (derivative magnitudes grow fast near `x = −1`)
  - `1/(1−x)` → `M = (n+1)! / (1 − x_hi)^(n+2)` (assuming `x_hi < 1`)
  - `1/(1+x²)` → use numeric: sample `f^(n+1)` at 200 points across the interval, take the max absolute value
- For custom expressions: numeric only — sample `f^(n+1)` at 200 points across `[x_lo, x_hi]` via the finite-difference core.
- Recompute `M` on interval drag end (not during drag — too expensive at high `n`).

### Tab C — Radius of convergence is real (DEFERRED)

**Visual:**
- Gold curve + polynomial stack from Tab A.
- **Convergence band:** translucent blue fill between `x = a − R` and `x = a + R`. Reuses `POSITIVE_GRADIENT` darkest stop at low opacity.
- **Divergence zones:** translucent red fill outside the band, both sides. Reuses `NEGATIVE_GRADIENT` darkest stop at low opacity.
- **Two dashed vertical lines** at `x = a ± R`, labeled `R = 1` (or whatever) at the top.
- Y-axis clipping (same as Tab A) so polynomial explosions visibly slam into the viewport edges.

**Interaction:**
- Order slider `n` (same as A).
- Function picker drives `R`. Default function: `1/(1+x²)`.
- ▶ play sweeps `n`.
- No interval handles, no center dragging.

**Math:**
- Per-preset `R` table:
  - `sin`, `cos`, `eˣ` → R = ∞ (don't draw the band; show "R = ∞" in overlay)
  - `ln(1+x)` → R = 1 (centered at 0)
  - `1/(1−x)` → R = 1 (centered at 0)
  - `1/(1+x²)` → R = 1 (centered at 0)
- For custom: show `R = ?` in overlay, no band. (Stretch goal: estimate via `1/limsup|aₙ|^(1/n)` from coefficients. Skip in v1.)

**Overlay:**
- Function + formula
- `R = 1` (or `∞` or `?`)
- `Center a = 0`
- Function-specific footnote. For `1/(1+x²)`: "Singularities at x = ±i (complex plane). Real axis looks fine, but the series can't cross them."

### Tab D — Maclaurin is Taylor at a=0 (DEFERRED)

**Visual:**
- Gold curve = true `f(x)`.
- **Single polynomial Pₙ** in pale lavender (no stack — would be visually noisy with `a` moving).
- **Anchor dot** at `(a, f(a))` — larger and more prominent than in A/B/C.
- Vertical dashed line at `x = a`.

**Interaction:**
- **`a` slider is the star** — wide range (function-appropriate, e.g. `−2π` to `2π` for trig), fine resolution, default 0.
- Order slider `n` secondary, default 5.
- ▶ play sweeps `a` left-to-right.
- **"Snap to a = 0" button** — animates `a` back to 0 over ~0.4s; overlay flashes "This is the Maclaurin series."

**Math:**
- Reuses Tab A's polynomial rendering and derivative core.
- For custom expressions, computing 20 derivatives many times per second during drag may lag. Mitigations: cap order at n=10 in Tab D, and/or recompute coefficients only every 3rd frame during drag. Decide once measured.
- For named presets: closed-form derivatives, essentially free.

**Overlay:**
- Function + formula
- `a = 1.57` (live)
- `n = 5`
- Polynomial expanded around `a` with numeric coefficients.
- When `|a| < ε`: append "← Maclaurin series" inline.

**Optional / deferred:** anchor trail (fading ghost of past positions). Skip in v1.

## Landing page change

`index.html` currently has two method cards (shell, washer). Add a third card "Taylor Series" linking to `series.html`. Use the same editorial style — same heading typography, same card layout, same screenshot-preview pattern (capture a screenshot of Tab A as `series-preview.png` after Tab A is built).

The landing-page card grid will need to handle three cards instead of two. If the existing two-up CSS uses a fixed `grid-template-columns: 1fr 1fr`, change it to `repeat(3, 1fr)` for desktop and let it stack to 1-up on mobile via existing breakpoint pattern.

## CLAUDE.md update

Add a new section to `CLAUDE.md` analogous to the existing shell architecture notes:

- The series page's mental model (chassis + four tabs).
- Where the derivative core lives.
- The `(a, n)` coefficient cache pattern.
- Note that Tabs C and D are spec'd in this design doc but not built — point the next session at this file.
- Update the file-tree section to reflect `series.html`, `series-preview.png`, and the `docs/superpowers/specs/` directory.

## Build order

1. **Scaffold `series.html`** by copying `washer.html` as the starting template (closer than `shell.html`). Strip washer-specific geometry. Set up `OrthographicCamera`, function-picker pill row with the new function library, tab bar A/B/C/D (only A and B wired up; C and D show "Coming soon" placeholder body).
2. **Numeric derivative core** + hardcoded derivative tables for named presets. Test against known values (e.g. `derivatives(sin, 0, 3) === -1`) before drawing anything.
3. **Polynomial evaluator** + coefficient caching.
4. **Tab A** — polynomial stack, order slider, play button, formula readout, anchor dot, dashed center line.
5. **Checkpoint:** verify with `sin(x)` over `[−2π, 2π]`, `n` ramping from 0 to 20. Polynomials should fan out from origin, stack should be visually clean. Pause and confirm the chassis carries weight before moving on.
6. **Tab B** — single polynomial + error band + Lagrange envelope + interval handles + numeric readouts. Verify `M` and bound math against a known case (`sin(x)`, `n=3`, `[−1, 1]` → bound easy to hand-check).
7. **Landing page** — third method card.
8. **`series-preview.png`** screenshot of Tab A for the card.
9. **CLAUDE.md update.**

## Risks & open questions

- **Custom-expression performance in Tab B:** computing the (n+1)th finite difference at 200 points may stutter for `n > 12`. Mitigation: cap order in Tab B at 12 for custom expressions only; named presets use closed-form M.
- **Numeric derivative noise:** finite differences degrade past about n=10 for typical step sizes. Acceptable for a visualization at this scope; will add a small "(approximate)" tag in the overlay when the custom branch is active.
- **Three.js dashed lines:** `LineDashedMaterial` requires `computeLineDistances()` and doesn't always look great. If the dashed envelope (Tab B) or convergence boundaries (Tab C) look bad, fall back to short tube segments with gaps.
- **Tab B error-band rendering:** the triangle-strip approach can flicker at zero-crossings. If it does, switch to two filled regions (above/below the curve) drawn separately.
- **Y-axis clipping** is critical for Tabs A and C — without it, polynomial blow-ups crush the rest of the scene to a flat strip. Implement the clip from the start, not as a fix later.

## Things to avoid

- Don't import math libraries (math.js, etc.). The numeric derivative + hardcoded preset table is enough.
- Don't add framework dependencies. Stay vanilla JS + Three.js, like the other pages.
- Don't auto-take `|f(x)|` anywhere — same lesson as the volumes pages: signs are pedagogical.
- Don't split into multiple files. The other pages are monolithic; series should be too.
- Don't add edge rings or seam decorations to the polynomial curves at high `n` — they'll moiré, same as in the shell page.
