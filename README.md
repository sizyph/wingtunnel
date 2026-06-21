# Wingtunnel

**Interactive 2D LBM airfoil simulator in a single HTML file.**

🛩  **Try it live:** **https://sizyph.github.io/wingtunnel/**

Or open [`wingtunnel.html`](wingtunnel.html) in a modern browser locally. No build step, no dependencies — everything runs in vanilla JavaScript on a 2D canvas.

## What it does

Wingtunnel solves the incompressible Navier–Stokes equations around a parametric body using the Lattice Boltzmann Method (D2Q9) with a Smagorinsky LES closure. You can:

- Choose the body — any **NACA 4-digit** code, a **Joukowski** airfoil, or a **cylinder**; optionally add a deflectable **trailing-edge flap**.
- Sweep angle of attack, freestream speed, and viscosity in real time — or **drag the wing** (and the flap) directly with the mouse.
- See aerodynamic coefficients (Cl, Cd, L/D, Re) computed live from momentum exchange on the wall, with a rolling **Cl/Cd sparkline**.
- Switch between **speed**, **vorticity**, **pressure**, **Lighthill noise**, and **temperature** visualizations.
- Drop **particle tracers** into the flow to see streamlines, separation, vortex shedding.
- Overlay lift / drag / chord vectors, a **Cp(x/c)** surface plot, and a **wake velocity profile**.
- Heat the wing and watch a buoyant plume form (**Boussinesq** thermal coupling).
- Release the wing into **free flight** — it pitches under its own aerodynamic moment.
- **Auto-sweep a polar** (Cl–α curve) and even **hear** the Kármán shedding tone.
- Run a built-in **shape optimizer** that perturbs the airfoil and AoA to maximize lift.
- Switch the collision operator between **BGK** and **MRT** for stability at low viscosity.
- Share any configuration via a **URL hash** — the whole scene round-trips.

## Quick start

```bash
git clone https://github.com/Sizyph/wingtunnel
cd wingtunnel
open wingtunnel.html   # or just double-click it
```

That's it. No npm, no Python, no server.

## Controls

| Key | Action | Key | Action |
|---|---|---|---|
| `space` | pause / resume | `g` | free flight (FSI) |
| `v` | cycle view | `t` | heated wing (Boussinesq) |
| `p` | particle tracers | `m` | MRT solver |
| `f` | force / Cp / wake overlays | `w` | free-slip walls |
| `o` | shape optimizer | `r` | reset flow field |

Mouse: **drag the wing** to set angle of attack, **drag the flap** to deflect it. All actions are also reachable from the floating panel on the right, including **Run Polar** and **Kármán tone**.

## What's inside

### Physics (LBM)

- **D2Q9 lattice**, 8 substeps per animation frame. Collision is selectable:
  - **BGK** (default): single relaxation time `ω = 1/(3ν + ½)`.
  - **MRT** (Lallemand–Luo): each moment relaxes at its own rate — stress modes at the Smagorinsky `ω`, bulk modes fixed (`s_e = s_ε = 1.4`, `s_q = 1.7`). Stable at much lower `ν` where BGK diverges.
- **Smagorinsky LES**: local relaxation augmented by `τ_eddy = ½(√(τ₀² + 18 C²|Π|/ρ) − τ₀)` with `C = 0.12`.
- **Zou–He velocity inlet** at the west boundary — reconstructs `f₁, f₅, f₈` analytically to enforce `(u_in, 0)`, density from `ρ = (f₀+f₂+f₄ + 2(f₃+f₆+f₇)) / (1−u_in)`. Mass-conserving.
- **Bouzidi (BFL) interpolated bounce-back** on the body. The wall fraction `q ∈ (0,1]` is computed per fluid–wall link via ray–polygon intersection whenever geometry changes:
  - `f_back = (f_k + (2q−1)·f_opp) / (2q)` for `q ≥ ½`
  - `f_back = 2q·f_k + (1−2q)·f₀(i_back, opp)` for `q < ½`
- **Channel walls**: no-slip (half-way bounce-back) by default, or **free-slip** (specular reflection of the wall-normal component) via toggle — removes the ~10 % blockage-induced Cl inflation.
- **Convective (Sommerfeld) outlet**: `f_end = (f_end_prev + U·f_pre)/(1+U)`. Shed vortices leave cleanly instead of reflecting upstream.
- **Boussinesq thermal coupling** (optional): a temperature scalar `T` is advected (upwind) and diffused (central) on the grid; buoyancy `F = (0, −g·β·(T−T∞))` enters the collision via Guo forcing. The wing is a hot Dirichlet boundary; inlet/walls are cold; outlet is zero-gradient.

### Geometry

- **NACA 4-digit**: standard thickness distribution (closed-trailing-edge coefficient `−0.1036`), mean camber line, cosine spacing. Pitched about the quarter-chord.
- **Joukowski**: a circle in the z-plane through `(1, 0)`, mapped via `w = z + 1/z`.
- **Cylinder**: the canonical Kármán-vortex-street bluff body.
- **Trailing-edge flap** (NACA mode): a second element at 30 % chord, deflection 0–45°, rotating about its hinge. Drag it directly or use the slider. Forces integrate over all elements; Cp(x/c) extends past the main chord to show it.

Re ≈ U · c / ν is displayed live.

### Diagnostics

- **Lift / drag / moment**: momentum exchange integrated over every wall link during bounce-back (`F_link = (f_pre + f_post)·e_k`), normalized by `½U²·c`. The pitching moment about the pivot drives free flight.
- **Cp(x/c) plot**: pressure coefficient along the surface, upper/lower split, auto-scaling x-range when a flap is present.
- **Wake profile**: `u(y)/U∞` one chord behind the body — the velocity defect is the drag (momentum theorem), an independent check on the wall-link Cd.
- **Polar sweep**: auto-ramps AoA from −10° to +25°, plotting the live Cl–α curve with the stall visible.
- **Cl/Cd sparkline**: rolling 240-frame history; oscillation = vortex shedding, flat = converged.
- **Kármán tone**: a brute-force DFT of the Cl history finds the dominant shedding frequency and maps it to an audible WebAudio sine.
- **Stability guard**: stride-sampled NaN + supersonic check each frame; auto-pauses with a warning.

### Visualization modes

- **Speed** `|u|` — Turbo colormap.
- **Vorticity** `ω = ∂v/∂x − ∂u/∂y` — diverging RdBu.
- **Pressure** `p = ρ/3 − p∞` — suction blob on the upper surface = lift.
- **Lighthill noise** `∂²(ρuᵢuⱼ)/∂xᵢ∂xⱼ` — the acoustic source term, log-magnitude diverging map.
- **Temperature** `T` — hot colormap (black → red → yellow → white), Boussinesq plume.

A canvas color legend (bottom-left) labels the active scale.

### Free flight (FSI)

The wing pitches under its own aerodynamic moment: `I·α̈ = M − γ·α̇`, integrated explicit-Euler once per frame. Grab the wing to steer; release and it weather-vanes toward trim and oscillates before damping out.

### Shape optimizer

Stochastic hill-climb over `(m, p, t, AoA)` for NACA airfoils: apply candidate → settle ~75 frames → average Cl over 35 frames → accept if better. AoA gets 45 % of perturbations. Unstable candidates are caught and rejected. ~30 iterations.

## Limitations

- **2D only.** No tip vortices, no induced drag, no spanwise effects.
- **Low Reynolds.** With a ~120-cell chord and `ν ∈ [0.005, 0.08]`, Re sits in the 10²–10³ range. Real flight is 10⁵–10⁷. Read relative trends across AoA and shape, not absolute Cl magnitudes. MRT + free-slip pushes the usable envelope but doesn't close this gap.
- **No transition modeling.** The flow is treated as turbulent everywhere; the laminar separation bubble at moderate Re is not captured.
- **Thermal is Boussinesq.** Buoyancy only — density is otherwise constant, so no compressible or large-ΔT effects.

## Possible extensions

- **Compressible / transonic LBM** to capture shocks on the upper surface near Mach 0.7.
- **Adjoint-based optimizer** for true `∂Cl/∂shape` gradients instead of hill-climbing.
- **Inverse design**: specify a target Cp(x/c), solve for the shape.
- **WebGL/WebGPU port**: one fragment shader per LBM substep → ~4× resolution at the same framerate, enough to resolve the boundary layer.
- **Multi-element / tandem** configurations (slat, biplane, formation).

## License

MIT.
