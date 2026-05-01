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

1. **PRESETS** — object mapping preset key → `{ label, formula, f, a, b, color, gradient }`. Adding a preset = adding an entry here + a button with `data-preset="key"`.
2. **`rebuild()`** — the central render function. Disposes the previous `stage` group, then:
   - draws the gold 2D curve as a `TubeGeometry` in the y=0 plane
   - draws the dashed z-axis (`LineDashedMaterial`)
   - draws shells: `[0, completedCount)` as full cylinders + the active shell with `sweepAngle`
3. **`makeShell(r, h, theta, gradient, isActive, idx, total)`** — returns a `Group` containing:
   - one `CylinderGeometry` with `thetaStart=0, thetaLength=theta, openEnded=true`, rotated so its axis is +z
   - color sampled from the preset's 3-stop gradient at `idx/(total-1)`
   - gold edge rings + seam lines, ONLY for the active shell (drawing them on every shell creates moiré at high density)
4. **Lighting** — `HemisphereLight` for fill + a `DirectionalLight` parented to the camera. The scene `add(camera)` call is necessary so the camera-light's transform inherits.
5. **Custom equation** — `compileExpression(str)` builds `new Function('x', 'with (Math) { return (...); }')`. Lowercases input and rewrites `^` → `**` and `π` → `pi`. Probes a few sample points to reject NaN/Infinity. The compiled function is stored on `PRESETS.custom.f`.

## Conventions

- **Math z-up:** `camera.up.set(0,0,1)`. Grid lies in xy plane (rotated `Math.PI/2` around X). Don't switch to default y-up — it'll invert everything.
- **Shells parameterize by radius**, not by index. `radii[i] = a + (b-a) * (i + 0.5) / N_SHELLS`.
- **Volume is computed numerically** (trapezoidal, 400 samples) inside `trapVolume(f, a, b)`. No closed-form per preset — just integrate.
- **Edge rings on inactive shells = visual noise.** If you ever want shell-separation back, prefer alternating shades or thicker gradient stops, not lines.
- **Active shell color** is `baseColor.multiplyScalar(1.35)` plus emissive — keeps it readable against the rest of the stack.
- **`always sweepAngle: number`** is in radians internally; the slider stores degrees and we convert.
- **At-max edge case:** when `shellCount === N_SHELLS`, the LAST shell becomes the "active" one so the sweep slider still works. See the `atMax` block in `rebuild()`.

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

## Things to avoid

- **Don't import packages.** No npm, no bundlers. Three.js comes from `unpkg` via the importmap. If you need a math library, add it the same way.
- **Don't use `eval`** outside of `compileExpression`. The `new Function` + `with (Math)` pattern is intentional — it's scoped, and rewriting `^` → `**` first makes power notation feel natural to kids.
- **Don't add framework dependencies** (React, Vue, etc.). The whole point is "open the file, it works."
- **Don't put colors in a stylesheet that the JS doesn't know about.** Three.js reads colors from the `PRESETS.gradient` triples. Keep them as `0x` hex literals.

## Common edits

- **Tweak shell density range:** `densitySlider` `min`/`max` attrs in HTML and the `Math.max(0.3, 8 / N_SHELLS)` floor in `play()`.
- **Different camera framing:** `frameCamera()`. Distance multiplier is `reach * 2.0 + 1.5` — bump down for tighter framing.
- **More expressive parser:** if a kid wants `|x|` (abs value) or `2x` (implicit multiply), extend `compileExpression`. Right now it's deliberately tiny.

## What hasn't been built but might come up

- **Washer method companion view** (disks perpendicular to axis instead of shells)
- **Side-by-side 2D and 3D view** showing the strip in 2D and the cylinder in 3D, linked
- **Show the volume sum building up** as shells are added — currently we just show the analytic integral
- **Rotating around the x-axis** instead of always y-axis — would need a "axis of rotation" toggle
