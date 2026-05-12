# Wingtunnel

**Interactive 2D LBM airfoil simulator in a single HTML file.**

🛩  **Try it live:** **https://sizyph.github.io/wingtunnel/**

Or open [`wingtunnel.html`](wingtunnel.html) in a modern browser locally. No build step, no dependencies — everything runs in vanilla JavaScript on a 2D canvas.

## What it does

Wingtunnel solves the incompressible Navier–Stokes equations around a parametric airfoil using the Lattice Boltzmann Method (D2Q9) with a Smagorinsky LES closure. You can:

- Choose the airfoil shape — any **NACA 4-digit** code, or a **Joukowski** parameterization.
- Sweep angle of attack, freestream speed, and viscosity in real time.
- See aerodynamic coefficients (Cl, Cd, L/D, Re) computed live from momentum exchange on the wall.
- Switch between **speed**, **vorticity**, and **pressure** visualizations.
- Drop **particle tracers** into the flow to see streamlines, separation, vortex shedding.
- Overlay the lift / drag / chord vectors on the airfoil.
- Run a built-in **shape optimizer** that perturbs the airfoil and AoA to maximize lift.

## Quick start

```bash
git clone https://github.com/Sizyph/wingtunnel
cd wingtunnel
open wingtunnel.html   # or just double-click it
```

That's it. No npm, no Python, no server.

## Controls

| Key | Action |
|---|---|
| `space` | pause / resume |
| `v` | cycle view (speed → vorticity → pressure) |
| `p` | toggle particle tracers |
| `f` | toggle force / chord overlay |
| `o` | start / stop shape optimizer |
| `r` | reset flow field |

All actions are also reachable from the floating panel on the right.

## What's inside

### Physics (LBM)

- **D2Q9 lattice** with BGK collision, 8 substeps per animation frame.
- **Smagorinsky LES**: local relaxation time augmented by `τ_eddy = ½(√(τ₀² + 18 C²|Π|/ρ) − τ₀)` with `C = 0.12`. Eddy viscosity rises in shear layers, keeps the wake stable at low molecular viscosity.
- **Zou–He velocity inlet** at the west boundary — reconstructs the unknown distributions `f₁, f₅, f₈` analytically to enforce `(u_in, 0)`, with density inferred from the closure `ρ = (f₀+f₂+f₄ + 2(f₃+f₆+f₇)) / (1−u_in)`. Mass-conserving, unlike a fixed `ρ=1` override.
- **Bouzidi (BFL) interpolated bounce-back** on the airfoil. The wall fraction `q ∈ (0,1]` is computed per fluid–airfoil link via ray–polygon intersection whenever geometry changes, then used in:
  - `f_back = (f_k + (2q−1)·f_opp) / (2q)` for `q ≥ ½`
  - `f_back = 2q·f_k + (1−2q)·f₀(i_back, opp)` for `q < ½`
- **Channel walls** use plain half-way bounce-back.
- **Outlet**: zero-gradient extrapolation (wave-reflective; see *Limitations*).

### Geometry

- **NACA 4-digit**: standard thickness distribution (closed-trailing-edge coefficient `−0.1036`), mean camber line, cosine spacing for finer LE/TE resolution. Pitched about the quarter-chord aerodynamic center.
- **Joukowski**: a circle in the z-plane through `(1, 0)`, mapped via `w = z + 1/z`.

Both produce polygons with a chord of ~120 grid cells. Re ≈ U · c / ν is displayed live.

### Diagnostics

- **Lift / drag**: momentum exchange integrated over every airfoil link during bounce-back (`F_link = (f_pre + f_post)·e_k`), normalized by `½U²·c`. Signed correctly for screen-y inversion. Smoothed with α = 0.95 EMA.
- **Chord**: max pairwise distance between polygon vertices (mode-agnostic).
- **Stability guard**: stride-sampled NaN + supersonic check each frame. Auto-pauses with a warning when either trips; cleared by Reset.

### Visualization modes

- **Speed**: `|u|` through a Turbo-style colormap.
- **Vorticity**: central-difference `ω = ∂v/∂x − ∂u/∂y` with a diverging RdBu colormap (CCW = red, CW = blue).
- **Pressure**: `p = ρ/3 − p∞` scaled so `|Cp| ≈ 2/contrast` saturates. Suction blob on the upper surface = lift.

A canvas-rendered color legend in the bottom-left labels the active scale.

### Particle tracers

~1500 passive markers, bilinear-interpolated velocity advection at 8× lattice time per frame, respawn at the inlet when they age out or hit the airfoil.

### Force / chord overlay

Dashed yellow chord line LE→TE, green lift arrow perpendicular to freestream, red drag arrow along it, both anchored at the quarter-chord. Drag arrow length is amplified 4× since Cd ≪ Cl.

### Shape optimizer

Stochastic hill-climb over `(m, p, t, AoA)` for NACA airfoils. Each trial:

1. Apply candidate, `initFluid()` for a clean field.
2. Settle for 75 frames (~600 LBM steps).
3. Average Cl over 35 frames.
4. Accept if better than the current best, otherwise revert.

AoA gets 45 % of perturbations (it has the largest single-step effect). Unstable candidates are caught by the stability guard and rejected automatically. Up to 30 iterations, ~1.7 s per iteration.

## Limitations

- **2D only.** Slice through the wing, not a full 3D simulation. No tip vortices, no induced drag, no spanwise effects.
- **Low Reynolds.** With ~120-cell chord and `ν ∈ [0.005, 0.08]`, Re sits in the 10²–10³ range. Real flight is 10⁵–10⁷. Don't read absolute Cl values as truth — relative behavior across AoA and shapes is meaningful, magnitudes are not.
- **Channel blockage.** The airfoil occupies ~15 % of channel height with no-slip top / bottom walls. This inflates measured Cl by roughly 10 %.
- **Outlet reflects.** Zero-gradient extrapolation lets vortices bounce back upstream. Visible in vorticity mode at high AoA — the wake never fully escapes.
- **No transition modeling.** The flow is treated as turbulent everywhere. Real airfoils have a laminar separation bubble at moderate Re that this won't capture.

## Possible extensions

If you want to fork and extend, the high-impact next steps are roughly:

- **Convective outlet** (`∂f/∂t + U ∂f/∂x = 0`) to kill upstream wake reflection.
- **Free-slip toggle** on channel walls for a freestream-ish setup.
- **Cp(x) curve** along the airfoil surface, the classic aero diagnostic.
- **WebGL/WebGPU port**: the whole D2Q9 update is one fragment shader and would unlock ~4× resolution at the same framerate.
- **Click-drag dye injection** for hand-painted streamlines.

## License

MIT.
