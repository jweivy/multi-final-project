# CLAUDE.md — Multi Final Project

Context for Claude Code when working in this repo.

## What this is

An interactive explorer for high-school multivariable / calc-2 topics: **shell method**, **washer method**, and **Taylor series** so far. Built as a final project; Julian + 3 teammates collaborate via GitHub.

- **Repo:** https://github.com/jweivy/multi-final-project (public)
- **Stack:** one HTML file per topic, vanilla JS, Three.js r160 from a CDN (`importmap`)
- **Lives at:** `C:\Users\jweiy\Documents\Claude-Projects\calc-volumes-3d`

## Project structure

```
calc-volumes-3d/
├── index.html        # landing page (links to the three method explorers)
├── shell.html        # shell method 3D explorer (the original)
├── washer.html       # washer method 3D explorer
├── series.html       # Taylor series 2D explorer (4 tabs, all built)
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
    └── launch.json   # local preview config (gitignored)
```

Each explorer is a single HTML file on purpose — no build step, no `npm`, runs from a `file://` open or any static server. Don't suggest splitting them up unless one grows past ~1500 lines.

## Architecture (mental model)

1. **Two top-level color constants:** `POSITIVE_GRADIENT` (blue → indigo → lavender) and `NEGATIVE_GRADIENT` (deep red → bright red). Every shell samples one of these by index. Per-preset gradients have been removed — change the look of the whole app in two lines.
2. **PRESETS** — object mapping preset key → `{ label, formula, f, a, b }`. Adding a preset = adding an entry here + a button with `data-preset="key"`. (Note: the `gradient` and `color` fields that used to live on each preset are gone. Don't add them back unless you have a reason — the simplification was deliberate.)
3. **`rebuild()`** — the central render function. Disposes the previous `stage` group, then:
   - draws the gold 2D curve as a `TubeGeometry` in the y=0 plane (handles negative z naturally)
   - draws the dashed z-axis from `zLo` to `zHi`, where these are the true vertical extents of `f` over `[a, b]` including any negative dip
   - draws shells: `[0, completedCount)` as full cylinders + the active shell with `sweepAngle`
   - sets `controls.target` to `(0, 0, midZ)` so the orbit center sits at the middle of the solid's vertical span
4. **`makeShell(r, h, theta, gradient, isActive, idx, total)`** — returns a `Group`:
   - `CylinderGeometry` with `thetaStart=0, thetaLength=theta, openEnded=true`, height = `Math.abs(h)`, then rotated so its axis is +z and translated by `h/2` (sign-correct: shell hangs below z=0 for negative h)
   - color sampled from the gradient at `idx/(total-1)`. **If `h < 0`, the function uses `NEGATIVE_GRADIENT` regardless of what `gradient` was passed in** — that's how the red overrides for sub-axis shells.
   - gold edge rings + seam lines, ONLY for the active shell (drawing them on every shell creates moiré at high density)
5. **Lighting** — `HemisphereLight` for sky/ground fill + a `DirectionalLight` parented to the camera. The `scene.add(camera)` call is necessary so the camera-light's transform inherits.
6. **Custom equation** — `compileExpression(str)` builds `new Function('x', 'with (Math) { return (...); }')`. Lowercases input and rewrites `^` → `**` and `π` → `pi`. Probes a few sample points to reject NaN/Infinity. The compiled function is stored on `PRESETS.custom.f`. Default value of `customExpr` is `sin(x)` over `[0, 2π]` — chosen so the +/- shell behavior is visible the moment a kid clicks the custom tab.

## Series page architecture (series.html)

A 2D Taylor/Maclaurin explorer with four tabs sharing one chassis. Spec lives at `docs/superpowers/specs/2026-05-02-taylor-series-explorer-design.md`; implementation plan at `docs/superpowers/plans/2026-05-02-taylor-series-explorer.md`.

1. **Same Three.js + importmap pattern** as shell/washer, but with `OrthographicCamera` looking down the +z axis so the xy plane reads as 2D. Same disposal/rebuild pattern (single `stage` group). Same color tokens. `OrbitControls` with `enableRotate = false` — pan and zoom only.
2. **Function library is different:** `sin`, `cos`, `exp`, `ln(1+x)`, `1/(1−x)`, `1/(1+x²)`, plus custom. Polynomial families like `√x` or `x²/2` are pedagogically boring here (their Taylor series is themselves).
3. **Derivative core (`derivAt`)** uses closed-form derivative formulas for the named presets (clean, no numeric noise). Custom expressions and `1/(1+x²)` fall back to recursive 5-point finite differences. Cap order at 20 — finite diff degrades past that.
4. **Coefficient cache (`coefficientsAt`)** memoizes `f^(k)(a)/k!` arrays keyed by `(presetKey, a, n)`. Polynomial evaluation across 400 plot points reuses the same coefficients. Cleared on custom-apply. LRU-bounded at 500 entries.
5. **Tab bar dispatches to four scene builders** (`buildTabA`, `buildTabB`, etc.) that each add geometry to the shared `stage` group. `rebuild()` disposes the stage and calls the right builder. Same pattern as the shell explorer's `rebuild`.
6. **Y-axis clipping in `clipY`** is critical — high-order polynomials shoot to ±∞ outside the convergence radius and would crush the rest of the scene. Clip everything to slightly outside the visible y-range so the polynomial visibly slams into the viewport edge.
7. **Halo trick for stroke weight:** Three.js `Line` ignores `linewidth` on most platforms, so the gold curve and active polynomial each get a translucent halo line drawn at a slight z-offset behind the crisp inner line. Cheap and effective.

**Tab status — all four shipped:**
- **A (polynomial stack):** order slider, play, live formula readout.
- **B (error bounds):** single `Pₙ` + actual error band + dashed Lagrange envelope + draggable interval handles + `M` / bound / actual readout.
- **C (convergence radius):** translucent blue convergence band over `[a−R, a+R]` and red divergence zones outside, dashed boundary lines. `R` hardcoded per preset (`R_TABLE`). Locks center to 0, hides the slider.
- **D (center point):** single `Pₙ` around live `a`, prominent anchor dot, dashed center line. `a` slider becomes the star. "Snap to a = 0" button does an instant snap + flashes a Maclaurin callout via Web Animations API. Play sweeps `a` left→right.

**Inline self-tests:** the derivative core and the polynomial evaluator each have an IIFE test block that runs at page load and logs PASS/FAIL to console. No test framework — these are `console.assert`-style. If you modify either module, reload and check the console for `[derivatives] all 21 tests passed` and `[taylor] all 11 tests passed`.

**Inspection hooks:** `window.__stage`, `window.__camera`, `window.__rebuild`, `window.__state()` are exposed for `preview_eval` debugging — harmless in production.

## Conventions

- **Math z-up:** `camera.up.set(0,0,1)`. Grid lies in xy plane (rotated `Math.PI/2` around X). Don't switch to default y-up — it'll invert everything.
- **Shells parameterize by radius**, not by index. `radii[i] = a + (b-a) * (i + 0.5) / N_SHELLS`.
- **Skip degenerate shells:** in `rebuild()`, `if (Math.abs(h) < 0.0005) continue;` — prevents flat zero-height geometry from glitching.
- **Volume is the *signed* integral** (trapezoidal, 400 samples). It can be negative if the function dips below the axis enough. The visual (red shells below, blue-purple above) makes the signed-ness obvious. Don't auto-take `|f(x)|` — the sign is the educational point.
- **Edge rings on inactive shells = visual noise.** Already removed. If you ever want shell-separation back, prefer alternating shades or thicker gradient stops, not lines.
- **Active shell color** is `baseColor.multiplyScalar(1.35)` plus emissive — keeps it readable against the rest of the stack.
- **`sweepAngle`** is in radians internally; the slider stores degrees and we convert.
- **At-max edge case:** when `shellCount === N_SHELLS`, the LAST shell becomes the "active" one so the sweep slider still has something to drive. See the `atMax` block in `rebuild()`.
- **Grid layout has 4 rows pinned by `grid-row`** in CSS. Don't rely on auto-placement — `#customForm` toggles `display: none`, and without explicit row numbers the scene-wrap shifts to row 2 and collapses to zero height. There's a comment in the CSS, but be careful adding new top-level children.
- **Camera framing samples 24 points across `[a, b]`** to find true `zMin/zMax` rather than just evaluating at endpoints. Necessary because functions like `sin` peak in the interior.

## Controls inventory (so future Claude knows what's there)

- **Preset pills:** `y = √x`, `y = x²/2`, `y = sin(x) + 0.2`, `✏️ custom`
- **Custom form** (visible only when `custom` is active): `y = <expr>`, `from x = <a>` `to <b>`, `apply` button
- **Three sliders** at the bottom: `Shell density` (4–60), `Completed shells` (0–N), `Active shell sweep angle` (0–360°)
- **Buttons stack:** `▶ play`, speed dropdown (slow/normal/fast/blitz), `↺ reset view`
- **Overlay** (top-left of canvas): function label, formula, signed volume, legend with 5 swatches

## Local dev

```bash
# preview server (matches the .claude/launch.json entry)
py -3.13 -m http.server 8765 --directory .
# open http://localhost:8765

# OR just open the file:
start index.html
```

For the Claude Code preview tool, the launch.json entry `volumes-3d` (port 8765) is configured at the trading-bot project level (`../trading-bot/.claude/launch.json`).

**Known quirk:** the preview screenshot tool sometimes hangs against a constantly-redrawing WebGL canvas (the `requestAnimationFrame` loop never idles). Don't take this as a sign the page is broken — `preview_eval` returning normally proves the page is alive. Just open the file in a real browser to verify visuals.

## Collaboration model

Public repo, three teammates added as direct collaborators (write access). Workflow:

```bash
git pull                     # before starting
git checkout -b your-feature # never push directly to main
# ...edit...
git push -u origin your-feature
# open a PR on GitHub, merge after a quick look
```

## Adding a new feature — checklist

When extending the app, in order of "least likely to break things":

1. **New preset:** add an entry to `PRESETS` + a button. That's it.
2. **New visual element:** add to `rebuild()` so it gets cleaned up by `disposeGroup(stage)` between rebuilds. Don't add directly to `scene` unless you also remove it manually (see how `axisLine` is handled).
3. **New slider/control:** add to the `.controls` grid in HTML, add an event listener that updates state and calls `rebuild()`. Update `syncLabels()` if it has a numeric readout.
4. **New input that affects the math:** route through the same path the custom-equation form uses — write to PRESETS, then `rebuild()` if the active preset changed.
5. **New top-level layout child** of `#app`: assign it `grid-row: N` explicitly. The auto-placement fallback breaks when other children toggle `display: none`.

## Things to avoid

- **Don't import packages.** No npm, no bundlers. Three.js comes from `unpkg` via the importmap. If you need a math library, add it the same way.
- **Don't use `eval`** outside of `compileExpression`. The `new Function` + `with (Math)` pattern is intentional — it's scoped, and rewriting `^` → `**` first makes power notation feel natural to kids.
- **Don't add framework dependencies** (React, Vue, etc.). The whole point is "open the file, it works."
- **Don't auto-take `|f(x)|` for volume.** The signed integral is intentional — the red shells visually pair with the negative number to teach what subtraction means in this context.

## Common edits

- **Recolor everything:** edit `POSITIVE_GRADIENT` and `NEGATIVE_GRADIENT` constants near the top of the script. The legend swatch is rebuilt from `POSITIVE_GRADIENT` inside `rebuild()`, so it stays in sync automatically.
- **Tweak shell density range:** `densitySlider` `min`/`max` attrs in HTML and the `Math.max(0.1, 8 / N_SHELLS)` floor in `play()`.
- **Different camera framing:** `frameCamera()`. Distance multiplier is `reach * 2.0 + 1.5` — bump down for tighter framing.
- **More expressive parser:** if a kid wants `|x|` (abs value) or `2x` (implicit multiply), extend `compileExpression`. Right now it's deliberately tiny.
- **Different play speeds:** the `<select id="speedSel">` options have multiplier values in seconds. Default normal = 1×, blitz = 0.15×.

---

## Note to next Claude session

(Picking this back up — read this section before doing anything substantive.)

**Status as of last edit:** Three explorer pages live (shell, washer, series). The series page has all four tabs implemented and verified — A (polynomial stack), B (error bounds), C (radius of convergence), D (center / Maclaurin). Landing page (`index.html`) has three method cards under the broadened title "Calculus, visualized."

**Pending — Julian still needs to give me three GitHub usernames** so I can fire the collaborator invites with `gh api -X PUT /repos/jweivy/multi-final-project/collaborators/USERNAME -f permission=push`. He hasn't shared them yet. Don't add anyone without confirmation. As soon as he sends the names, the command above does it. The active gh auth is `jweivy` (verified via `gh auth status`).

**Likely next requests** (in rough order of probability):
1. **Refresh `series-preview.png`** — the current preview was generated by PIL with synthetic geometry rather than captured from the live page. With four tabs now live, a real screenshot from the canvas would read better.
2. **Annotate shells/washers with their individual contribution** to the integral (e.g., floating "+0.23" labels per shell).
3. **Toggle for absolute volume vs signed volume** on the shell/washer pages.
4. **2D companion plot** on the shell page showing the curve in the xz plane with the active shell's strip highlighted.
5. **x-axis revolution toggle** for shells (more invasive — changes the coordinate setup).
6. **Numerical estimate of R** for custom expressions in Tab C via `1/limsup |aₙ|^(1/n)` from the cached coefficients (currently shows `?`).

**If a kid asks about TeX-style math input** (`\sin`, `\frac{...}`): none of the parsers can handle that. Would need to either guide them to JS-style syntax or import a math expression parser like math.js (acceptable since it's a single CDN line).

**Don't suggest splitting any of the .html files.** Each page is monolithic on purpose. If one grows past ~1500 lines, that conversation can happen — until then, keep them monolithic.

**Watch for:**
- Any change to `rebuild()` in `shell.html` should be tested with the `custom` preset set to `sin(x)` on `[0, 2π]` — exercises positive + negative shells, the at-max edge case, and camera framing for negative z.
- Any change to `derivAt` or `coefficientsAt` in `series.html` should be tested by reloading and checking the console for the two PASS lines from the inline self-tests.
- Any change to the series-page polynomial stack rendering should be tested with `1/(1+x²)` because that's the function that exercises the y-clipping at the convergence-radius edge.
