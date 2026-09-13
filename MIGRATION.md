# Consumer wiring

This repo is the extracted renderer. Do **not** edit `qga_engine` or
`inner_cone` from this extract. [`inner_cone`](https://github.com/kinaar8340/inner_cone)
@ `89a890c` git-pins `qga_engine@7e7866b` + `qga_gpu@b9c9994`.
[`qga_engine`](https://github.com/qga-lab/qga_engine) visual tip is `7e7866b`.
No `v0.1.0` tags. Do not float `main`.

## Status (Software fact)

| Consumer | Form | Features | Fiber conversion |
|----------|------|----------|------------------|
| `inner_cone` @ `89a890c` | git `qga_engine@7e7866b` + `qga_gpu@b9c9994` | `["capture"]` | `geometry::gpu_fiber` |
| `qga_engine` (`qga-app`) @ `7e7866b` | git, `rev = "b9c9994"` | `winit`, `headless`, `capture`, `glow` | `qga-app::convert` |

`qga-math` / `qga-sim` stay in the engine workspace. Do **not** enable
`qga-gpu/qga-math` on either consumer: the renderer must not take a default
math dep.

## inner_cone

Git consumer (`89a890c`). Not a sibling path:

```toml
qga-math = { git = "https://github.com/qga-lab/qga_engine", rev = "7e7866bb2611d4ab35e6e3c1f46a3f8dd9b4320d" }
qga-sim  = { git = "https://github.com/qga-lab/qga_engine", rev = "7e7866bb2611d4ab35e6e3c1f46a3f8dd9b4320d" }
qga-gpu  = { git = "https://github.com/qga-lab/qga_gpu", rev = "b9c999406e266a585769a4804c6968de0e6d2237", features = ["capture"] }
```

`capture` is the right feature set for `--export` / F12. `winit` is this
crate’s default; `glow` is optional bloom.

`inner_cone` has no headless binary. `--export --frames N` is the windowed
stand-in and does not print or assert `UploadStats`. Until it does (print
from `--export`, or a thin `--headless --frames N` that exits non-zero on
the predicates below), this crate’s demo is the only proof the extracted
renderer is not retessellating the sculpture every frame:

```bash
make headless   # 8 offscreen frames; requires static_uploads == 1
make ring       # 300 dirty frames; ring_copies + particle_fallbacks >= frames
```

Fallbacks are allowed and counted. `particle_skipped == 0` when dirty.
This 4090 (`f263ea7`): 8 still `static_uploads=1`; 300 dirty
`ring_copies=301`, `particle_fallbacks=0`, `particle_grows=0`. That proves
the demo sculpture is not retessellated. It does **not** prove `inner_cone`
mosaic / hull / live harmonics or `qga-app` lab / realm / cosmos / oam /
reveal — those binaries still do not print `UploadStats`.

## qga_engine

Published: [`qga-lab/qga_engine`](https://github.com/qga-lab/qga_engine)
@ `7e7866b`. In-tree `crates/qga-gpu` is gone. Workspace dep:

```toml
# qga_engine/Cargo.toml workspace.dependencies
qga-gpu = { git = "https://github.com/qga-lab/qga_gpu", rev = "b9c999406e266a585769a4804c6968de0e6d2237", features = ["winit", "headless", "capture", "glow"] }
```

The pin is workspace `rev = "b9c9994"`. `Cargo.lock` is a lockfile, not the
contract. Do not bump this sha from a README commit. `inner_cone` git-pins
the same two revs. No `v0.1.0`.

Engine docs at `7e7866b` (README, DESIGN, AGENTS, Makefile, `docs/SCENES.md`)
match this split: this crate owns the frame; `qga-app` owns lab / realm /
cosmos / oam / reveal. Software fact of that tree: cosmos default 262 144
bodies (cap 524 288), realm 128 × 128 fibers and 256² terrain. Palettes
rewrite `GpuParticle` hue in `qga-app::convert`. Space on cosmos hides HUD
tabs; it does not pause. `--preset`, `--dump-species`, and `$QGA_PLAYGROUND`
are engine CLI. `qga-app --headless` does not print or assert `UploadStats`.

Do not open a PR into `qga_engine` from this extract.

## Call-site notes (Software fact)

| Engine / inner_cone names | This crate |
|---------------------------|------------|
| `update_static_fibers` | `retain_static_fibers` (alias kept) |
| `update_solid_fibers` | `write_live_fibers` (alias kept) |
| `upload_gpu_hubs(&[(Vec3, f32, Vec3)])` | `upload_hubs(&[GpuHub])` (tuple alias kept) |
| `write_hud_verts` | `write_hud` (alias kept) |
| `write_particles(&[qga_sim::Particle])` | `write_particles(&[GpuParticle])` |
| `update_solid_fibers(&[qga_math::Fiber])` | `GpuFiber` or `--features qga-math` `From` |
| `render(..., SceneKind::Reveal, time, grab)` | `render(gpu, cam, vis, time, capture)` |

`inner_cone` `03e1fb2` already calls the new names. Static topology: tessellate
once, then `retain_meshes` / `retain_static_fibers`. Identical re-upload is a
no-op (`static_uploads` stays 1; `static_skipped` grows). Live harmonics go
through `write_live_fibers`. Particles through the staging ring.
