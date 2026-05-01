# Multi Final Project — Shell Method 3D Explorer

An interactive 3D visualization of the **shell method** for computing volumes of revolution. Built with [Three.js](https://threejs.org/) and runs entirely in the browser.

> Pick a function. Drag the sliders. Watch each cylindrical shell wrap around the z-axis and stack up to form the solid of revolution.

## Live demo

Open `index.html` in any modern browser — no install, no server needed.

## Features

- **Three preset functions:** `y = √x`, `y = x² / 2`, `y = sin(x) + 0.2`
- **Drag-to-rotate 3D scene** (mouse drag · scroll-zoom · right-click pan)
- **Three sliders:**
  - **Shell density** (4 → 60) — how finely the solid is approximated
  - **Completed shells** — how many fully wrapped shells are visible
  - **Active shell sweep angle** (0° → 360°) — sweeps a single shell from a thin strip into a full cylinder, showing the "wrap" action
- **Gradient color per shell** — inner-to-outer color ramp makes the depth obvious; each preset has its own palette
- **Auto-play button** — animates each shell sweeping into place, one at a time
- **Live volume calculation** — numerically integrates `∫ 2π · x · f(x) dx` and shows the result in the overlay

## How the math maps to the geometry

For each radius *r* in the integration interval `[a, b]`:

- The **2D curve** `y = f(x)` lives in the plane `y = 0` (drawn in gold)
- A **cylindrical shell** at radius *r* has:
  - **circumference** `2π · r`
  - **height** `f(r)`
  - **infinitesimal thickness** `dr`
- Its volume is `2π · r · f(r) · dr`
- Integrating from `a` to `b` gives the total volume of the solid

The viewer renders each shell as a partial cylinder (`THREE.CylinderGeometry` with `thetaStart` / `thetaLength` for the sweep). The "active" shell carries gold edges so you can see where the wrap starts and stops.

## Running locally

The page is one self-contained file that loads Three.js from a CDN. Just open it:

```bash
# any of these works:
open index.html              # macOS
xdg-open index.html          # Linux
start index.html             # Windows
```

If you want a local server (e.g. for hot-reload tooling), any static server works:

```bash
python -m http.server 8765
# → http://localhost:8765
```

## Files

```
.
├── index.html        # the entire app — Three.js + UI + math
├── README.md         # this file
└── .gitignore
```

## Tech notes

- Three.js `r160` via the [unpkg](https://unpkg.com) CDN with an `importmap`
- Hemisphere + camera-anchored directional lights so no face goes black on rotation
- Volume integration via trapezoidal rule (400 sample points)
- Gradient sampled from a 3-stop palette per preset; inner shells use the start of the ramp, outer shells use the end

## License

MIT
