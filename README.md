# Wingtunnel

**Interactive 2D LBM airfoil simulator in a single HTML file — GPU-accelerated.**

🛩  **Try it live:** **https://sizyph.github.io/wingtunnel/**

Or open [`wingtunnel.html`](wingtunnel.html) in a modern browser locally. No build step, no dependencies, no server — one self-contained HTML file. The fluid solve runs on the GPU (WebGL2) by default, with a complete CPU fallback.

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
- Run the whole thing on the **GPU at up to 2560×1280**, or fall back to the CPU solver.
- Share any configuration via a **URL hash** — the whole scene round-trips.

## Quick start

```bash
git clone https://github.com/Sizyph/wingtunnel
cd wingtunnel
open wingtunnel.html   # or just double-click it
```

That's it. No npm, no Python, no server.

## Solver: GPU by default, CPU fallback

Two complete D2Q9 solvers share the same physics and UI:

- **GPU (WebGL2)** — the default. Distributions live in three ping-ponged `RGBA32F` textures (`f0–3`, `f4–7`, `f8/ρ/uₓ/u_y`); a fragment shader does pull-streaming + bounce-back + collision (BGK or MRT) + Smagorinsky + Boussinesq buoyancy. Temperature is a separate ping-pong field; particles are advected in a position texture and drawn as point sprites; forces come from a per-cell momentum-exchange pass reduced to a single texel (1-pixel readback); Cp/wake read a sim-resolution downsample of the field. Resolution is selectable **768×384 → 2560×1280**, with substeps auto-scaled to hold framerate.
- **CPU** — the fallback and reference. Same boundary conditions and collision; runs at 400×200. Used automatically when WebGL2 or float-texture support is missing, and reachable any time via the **⚡ GPU high-res** toggle or `#solver=cpu`.

Geometry (NACA/Joukowski/cylinder/flap rasterization, chord, LE/TE) is computed on the CPU in both modes and uploaded to the GPU as a high-resolution mask, so the airfoil is crisp at full GPU resolution.

The active solver is encoded in the share hash (`solver=gpu|cpu`), so links keep their mode.

## Controls

| Key | Action | Key | Action |
|---|---|---|---|
| `space` | pause / resume | `g` | free flight (FSI) |
| `v` | cycle view | `t` | heated wing (Boussinesq) |
| `p` | particle tracers | `m` | MRT solver |
| `f` | force / Cp / wake overlays | `w` | free-slip walls (CPU) |
| `o` | shape optimizer | `r` | reset flow field |

Mouse: **drag the wing** to set angle of attack, **drag the flap** to deflect it. All actions are also reachable from the floating panel on the right, including the GPU toggle, resolution, **Run Polar**, and **Kármán tone**.

## What's inside

### Physics (LBM)

- **D2Q9 lattice**, multiple substeps per animation frame. Collision is selectable:
  - **BGK** (default): single relaxation time `ω = 1/(3ν + ½)`.
  - **MRT** (Lallemand–Luo): each moment relaxes at its own rate — stress modes at the Smagorinsky `ω`, bulk modes fixed (`s_e = s_ε = 1.4`, `s_q = 1.7`). Stable at much lower `ν` where BGK diverges. Implemented identically on CPU and in the GPU step shader.
- **Smagorinsky LES**: local relaxation augmented by `τ_eddy = ½(√(τ₀² + 18 C²|Π|/ρ) − τ₀)` with `C = 0.12`.
- **Zou–He velocity inlet** (CPU) / equilibrium-velocity inlet (GPU) at the west boundary.
- **Bouzidi (BFL) interpolated bounce-back** on the body (CPU); the GPU uses half-way bounce-back against the high-res mask.
- **Channel walls**: no-slip by default, or **free-slip** (specular reflection) via toggle (CPU) — removes the ~10 % blockage-induced Cl inflation.
- **Convective (Sommerfeld) outlet**: shed vortices leave cleanly instead of reflecting upstream.
- **Boussinesq thermal coupling** (optional): a temperature scalar `T` is advected (upwind) and diffused (central); buoyancy `F = (0, −g·β·(T−T∞))` enters the collision via Guo forcing. Hot wing, cold inlet/walls, zero-gradient outlet. Runs on both CPU and GPU.

### Geometry

- **NACA 4-digit**: standard thickness distribution (closed-trailing-edge coefficient `−0.1036`), mean camber line, cosine spacing. Pitched about the quarter-chord.
- **Joukowski**: a circle in the z-plane through `(1, 0)`, mapped via `w = z + 1/z`.
- **Cylinder**: the canonical Kármán-vortex-street bluff body.
- **Trailing-edge flap** (NACA mode): a second element at 30 % chord, deflection 0–45°, rotating about its hinge. Drag it directly or use the slider. Forces integrate over all elements.

Re ≈ U · c / ν is displayed live.

### Diagnostics

- **Lift / drag / moment**: momentum exchange over every wall link (`F_link ≈ 2·f_a·e_a`), normalized by `½U²·c`. On the GPU, a per-cell pass + log-depth parallel reduction sums it to a single texel read back per few frames. The pitching moment drives free flight.
- **Cp(x/c) plot**: pressure coefficient along the surface, upper/lower split, auto-scaling x-range when a flap is present. In GPU mode, classified from a field-consistent downsampled mask.
- **Wake profile**: `u(y)/U∞` one chord behind the body — the velocity defect is the drag (momentum theorem).
- **Polar sweep**: auto-ramps AoA from −10° to +25°, plotting the live Cl–α curve with the stall visible.
- **Cl/Cd sparkline**: rolling 240-frame history; oscillation = vortex shedding, flat = converged.
- **Kármán tone** (CPU): a DFT of the Cl history maps the dominant shedding frequency to an audible WebAudio sine.

### Visualization modes

- **Speed** `|u|` — Turbo colormap.
- **Vorticity** `ω = ∂v/∂x − ∂u/∂y` — diverging RdBu.
- **Pressure** `p = ρ/3 − p∞` — suction blob on the upper surface = lift.
- **Lighthill noise** `∂²(ρuᵢuⱼ)/∂xᵢ∂xⱼ` (CPU) — the acoustic source term, log-magnitude diverging map.
- **Temperature** `T` — hot colormap, Boussinesq plume.

A canvas color legend (bottom-left) labels the active scale.

### Free flight (FSI)

The wing pitches under its own aerodynamic moment: `I·α̈ = M − γ·α̇`, integrated once per frame. Grab the wing to steer; release and it weather-vanes toward trim and oscillates before damping out.

### Shape optimizer

Stochastic hill-climb over `(m, p, t, AoA)` for NACA airfoils: apply candidate → settle → average Cl → accept if better. AoA gets 45 % of perturbations. ~30 iterations. Runs against whichever solver is active.

## Limitations

- **2D only.** No tip vortices, no induced drag, no spanwise effects.
- **Low Reynolds.** Re sits in the 10²–10⁴ range (higher on the GPU's finer grid). Real flight is 10⁵–10⁷. Read relative trends across AoA and shape, not absolute Cl magnitudes.
- **No transition modeling.** The flow is treated as turbulent everywhere; the laminar separation bubble at moderate Re is not captured.
- **Thermal is Boussinesq.** Buoyancy only — density is otherwise constant, so no compressible or large-ΔT effects.
- **A few features are CPU-only:** free-slip walls, the Lighthill-noise view, and the Kármán tone. Toggle off the GPU (or `#solver=cpu`) to use them.

## Possible extensions

- **Compressible / transonic LBM** to capture shocks on the upper surface near Mach 0.7.
- **Adjoint-based optimizer** for true `∂Cl/∂shape` gradients instead of hill-climbing.
- **Inverse design**: specify a target Cp(x/c), solve for the shape.
- **Local grid refinement** (multi-block LBM) for fine-near-foil / coarse-far resolution.
- **Port the last CPU-only bits to the GPU** (free-slip walls, noise view, tone).

## License

MIT.
