# CLAUDE.md — Multi Final Project

Context for Claude Code when working in this repo.

## What this is

An interactive 3D explorer of the **shell method** for volumes of revolution. Built for a high-school multivariable / calc-2 final project. Julian + 3 teammates collaborate on it via GitHub.

- **Repo:** https://github.com/jweivy/multi-final-project (public)
- **Stack:** single `index.html`, vanilla JS, Three.js r160 from a CDN (`importmap`)
- **Lives at:** `C:\Users\jweiy\Documents\Claude-Projects\calc-volumes-3d`

## Project structure

```
calc-volumes-3d/
├── index.html        # the entire app (UI + Three.js scene + math)
├── README.md         # public-facing project description
├── CLAUDE.md         # this file
├── .gitignore        # ignores .claude/ and OS junk
└── .claude/
    └── launch.json   # local preview config (gitignored)
```

Everything is in `index.html` on purpose — no build step, no `npm`, runs from a `file://` open or any static server.

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

**Status as of last edit:** All four shell-method features are working — three preset functions, a custom-equation tab with a parser, density / completed / sweep sliders, play with speed selector, and red-shell rendering for negative function values. Most recent push: commit `b0a3df3`, "Unify all positive shells under one blue-purple gradient." Repo is at `https://github.com/jweivy/multi-final-project`, public, on branch `main`.

**Pending — Julian needs to give me three GitHub usernames** so I can fire the collaborator invites with `gh api -X PUT /repos/jweivy/multi-final-project/collaborators/USERNAME -f permission=push`. He hasn't shared them yet. Don't add anyone without confirmation. As soon as he sends the names, the command above does it. Pick the right account scope — the active gh auth is `jweivy` (verified via `gh auth status`).

**Likely next requests** (in rough order of probability):
1. **Washer method companion view.** Disks perpendicular to the axis instead of shells. Either a toggle on the existing page or a separate section. Same `f, a, b` inputs, different geometry. The volume formula switches to `∫ π · f(x)² dx` and the integration direction may swap (around x-axis instead of z-axis).
2. **Annotate shells with their individual contribution** (e.g., a small floating label "+0.23" or "-0.18" near each shell). Would teach kids what each strip is worth in the integral.
3. **Toggle for absolute volume vs signed volume.** Right now we always show signed. A small switch in the overlay would let them flip and see `∫ 2π·x·|f(x)| dx`.
4. **2D companion plot** — a small inset showing the curve in the xz plane with a vertical strip highlighted at the same `r` as the active shell. Helps connect the 2D math to the 3D solid.
5. **x-axis revolution** instead of z-axis — a "rotate around" toggle. This is more invasive (changes the whole coordinate setup), so probably last.

**If a kid asks about TeX-style math input** (`\sin`, `\frac{...}`): the parser as built can't handle that. Would need to either guide them to JS-style syntax or import a math expression parser like math.js (acceptable since it's a single CDN line).

**Don't suggest splitting `index.html` into multiple files.** I considered it; the single-file simplicity is the point. If the file grows past ~1500 lines, that conversation can happen — until then, keep it monolithic.

**Watch for:** any change that touches `rebuild()` should be tested with the `custom` preset set to `sin(x)` on `[0, 2π]` because that's the configuration that exercises both positive and negative shells, the at-max edge case, and the camera framing for negative z. If that scenario looks right, most others will too.
