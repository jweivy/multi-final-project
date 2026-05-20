# Overnight Build — Shared Conventions

You are building ONE new self-contained HTML explorer file at the repo root. Match the existing pages so the suite feels coherent.

## Hard constraints

- **No build step, no npm, no bundlers.** Single HTML file, vanilla JS.
- **Three.js r160 from unpkg** via importmap (copy the block from `contour.html` or `shell.html`).
- **KaTeX 0.16.9 from jsdelivr** via `<link>` + `<script defer>` (copy from `contour.html`).
- **Don't import any other libraries** unless absolutely necessary. If you need a library, add it the same way (CDN link).
- **File is self-contained.** All CSS/JS inline in the one HTML file. No external assets except the CDNs above.

## Audience and tone

The audience is **high-school students seeing this concept for the first time**.

- Math must look like real math. Stacked fractions, proper exponents (use KaTeX). No `x^3` in display text — `x³` or KaTeX. No `5.79e-7` — `5.79 × 10⁻⁷`.
- Plain language beats correct jargon in labels and captions. Save the technical name for after the picture has made the idea obvious.
- Show concrete numbers, not abstractions. "These two arrows point the same way, so the dot product is +1.85" beats "the angle is acute."
- **One click should do something.** A slider should produce a visible change immediately.
- No invisible state. If the visual changes because of a preset auto-switch or constraint, tell the user with a caption.

## Aesthetic (match `contour.html` exactly)

CSS color tokens (use these exact names and values):

```css
:root {
  --bg: #0e1116;
  --ink: #e8e6df;
  --soft: #7a8794;
  --gold: #ffd166;
  --blue: #6aa9ff;
  --rose: #ff7eb6;
  --pill: #15181d;
  --pill-border: #23272d;
}
```

Typography: `font-family: Georgia, "Source Serif Pro", serif;` on body. Same as contour.

Layout pattern (copy from contour.html):
- `#app` grid with rows: header / customForm (optional) / scene-wrap (1fr) / footer
- **Pin every direct child of `#app` with explicit `grid-row: N`** — when `#customForm` toggles `display:none`, auto-placement collapses the scene to 1px otherwise. There is a comment in contour.html explaining this. Respect it.
- Header has: home link back to `index.html` (uses `.home-link` class from contour), title with gold accent, preset pills (`.presets button`)
- Overlay box top-right of canvas with `.overlay` styling from contour.html — title, formula (KaTeX), readouts, legend

## Three.js conventions

- `importmap` with `three` → `https://unpkg.com/three@0.160.0/build/three.module.js`, `three/addons/` → `https://unpkg.com/three@0.160.0/examples/jsm/`
- For 3D vector scenes: PerspectiveCamera, OrbitControls, optional `camera.up.set(0,0,1)` if you want z-up math style
- For 2D vector-field scenes: prefer plain `<canvas>` 2D context. Three.js with an OrthographicCamera is fine too — match what `series.html` does.
- **Disposal pattern:** keep all dynamic geometry in a single `stage` group. On rebuild, call `disposeGroup(stage)` (recurse, dispose geometries + materials) then re-add. See shell.html for the helper.
- Expose `window.__stage`, `window.__camera`, `window.__rebuild`, `window.__state()` for debugging (harmless in production, lets the user inspect via console).
- Inline self-test: at end of script, run a tiny IIFE that asserts a few invariants of your math code (e.g., cross product orthogonality, divergence of `(x,y)` equals 2) and `console.log("[<feature>] all N tests passed")`. If a test fails, log it loudly.

## Math display

- Use KaTeX for any equation that contains fractions, exponents, vectors, or Greek letters. `katex.render(latex, element, { throwOnError: false })`.
- For inline numbers in HTML readouts, write a `prettyNumber(v)` helper: fixed notation between 0.001 and 9999, else `n × 10ᵏ` with HTML `<sup>` for the exponent. **Never show bare `1.73e-7` in the UI.**
- For arrow notation, use `\vec{a}` in KaTeX or `**a**` styled with bold + serif italic in plain HTML.

## Navigation

Every page must include a `.home-link` (the gray pill in the header) that goes to `index.html`. Style it like contour.html does.

## What NOT to do

- Don't modify `index.html` — a separate step will wire your new page in.
- Don't modify any other `.html` file.
- Don't add framework dependencies (React, Vue, etc.).
- Don't `eval` user input. If you accept a custom expression, use the `new Function('x', 'with (Math) { return (...); }')` pattern from `shell.html`.
- Don't show raw scientific notation, raw `x^n`, or jargon in user-facing strings.
- Don't push to git. Don't run `git commit`. Just write the file.

## Verification self-check before reporting done

1. Open your new HTML file in your head. Does it load? Are all CDN URLs correct?
2. Does the home link go to `index.html`?
3. Did you pin every `#app > *` direct child with `grid-row`?
4. Does at least one preset render *something interesting* on first load (don't require the user to fiddle)?
5. Does the inline self-test pass?
6. Is the math display using KaTeX (or proper Unicode), not `^` notation?

Report what you built in 5 bullets max.
