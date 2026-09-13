![qga-gpu](bg1_qga_gpu.jpg)

# qga-gpu

Spine: [`qga`](https://github.com/qga-lab/qga) — manuscript + pedagogical Python  
Shared math: [`flux_hopf_lib`](https://github.com/qga-lab/flux_hopf_lib)  
Engine: [`qga_engine`](https://github.com/qga-lab/qga_engine) (scenes, Rust math) · this repo (frame)  
This repo: wgpu/Vulkan frame, pipelines, upload stats. Not world meaning.

wgpu/Vulkan renderer. **This crate owns the frame.** Geometry meaning lives in
[`qga_engine`](https://github.com/qga-lab/qga_engine) (`qga-math` / `qga-sim`
/ `qga-app`) and the manuscript [`qga`](https://github.com/qga-lab/qga).

Renderer claims are **Software fact**. Geometry in the public demo is **Model**.
Binaries print `claims=Software fact` via `qga_gpu::print_claim_banner`.

## Ten minutes

```bash
git clone https://github.com/qga-lab/qga_gpu
cd qga_gpu
make demo
```

That is the public demo: a 64×64 speaker lattice (instanced geodesic orbs +
one live ring per cell), a glass fabric, and a **65 536-particle** rainbow
ocean. Esc (or close the window) prints `UploadStats`. LMB orbit, wheel zoom,
`C` cinematic, `G` glow, Space pause.

Look at the last lines:

```
done scene=gradient … grid=64 orbs=4096 rings=4096 … particles=65536
write_buffer=… ring_copies=… static_uploads=1 … particle_skipped=0
  particle_grows=… particle_fallbacks=…
claims=Software fact  not_a_proof_of=inner_cone mosaic / qga-app cosmos
```

Dirty particles: `particle_skipped == 0` and
`ring_copies + particle_fallbacks >= frames`. `static_uploads == 1` means the
static topology was written once. Fallbacks are counted, not fatal.

No window:

```bash
make demo-headless   # 30 frames, --no-capture; same counters, asserts on exit
```

4k sculpture smoke (not the public demo):

```bash
make demo-tiny       # 1 sphere + 2 cones + torus + 4k particles
make ring            # 300 dirty 4k frames, headless
```

Those counters prove **this crate’s** upload path. They do **not** prove
`inner_cone` mosaic/hull or `qga-app` cosmos. `inner_cone` does not print
`UploadStats`. Neither does `qga-app --headless`.

The public demo has **no path to `qga_engine`**. Fiber conversion stays in
the consumer (`inner_cone` `geometry::gpu_fiber`). There is no sibling
`qga-math` path dep.

## Pin this crate

A stranger should pin a published sha, not float `main`. There is no
`v0.1.0` tag.

```toml
qga-gpu = { git = "https://github.com/qga-lab/qga_gpu", rev = "b9c999406e266a585769a4804c6968de0e6d2237" }
```

[`qga_engine`](https://github.com/qga-lab/qga_engine) @ `7e7866b`
git-depends with `rev = "b9c9994"` (`features = ["winit", "headless",
"capture", "glow"]`). `Cargo.lock` is not the pin; the workspace `rev`
is. [`inner_cone`](https://github.com/kinaar8340/inner_cone) @ `89a890c`
is a **git** consumer of `qga_engine@7e7866b` + this sha (`features =
["capture"]`). No sibling `../` path. See [MIGRATION.md](MIGRATION.md).

## Hardware targets

Three targets. The 4090 table is the **lab**, not “this box.”

| Target | What to run | Notes |
|--------|-------------|--------|
| Laptop demo | `make demo-tiny` | 1 sphere + 2 cones + 4k particles |
| 4090 lab | `make demo` / `make bench*` | 64×64 speakers + 65 536 motes; counters below |
| Headless CI | `make demo-headless` / `make demo-tiny --headless` | Skip if no Vulkan adapter |

**Lab (4090)**

| | |
|---|---|
| GPU | NVIDIA GeForce RTX 4090, 24 GiB, driver 580, Vulkan 1.4 |
| CPU | AMD Ryzen 9 3900X, 12 cores / 24 threads |
| OS | Ubuntu, GNOME, **Wayland** |

Vulkan through `wgpu` only. No OpenGL. CUDA is out of scope for v0. `make demo`
is sized for the lab (`--grid 64 --fluid` → 4096 speakers + 65 536 motes).

## Workspace

```
crates/qga-gpu         library (WGSL in crates/qga-gpu/src/shaders/)
crates/qga-gpu-demo    4k sculpture smoke (`make demo-tiny`)
crates/qga-gpu-bench   public 65k ocean + Hopf / hold / loom / core-loom bench
```

Extracted from `qga_engine/crates/qga-gpu` and the inner_cone upload path.

## Build

```bash
make check                   # cargo check --workspace && cargo test -p qga-gpu
make demo                    # public 65k ocean, until Esc, prints UploadStats
make demo-headless           # 30 offscreen 65k frames; asserts counters
make demo-tiny               # 4k sculpture window
make headless                # 8 offscreen 4k frames; static_uploads == 1
make ring                    # 300 dirty 4k frames, headless (in-flight map_async)
make ring-windowed           # 300 dirty 4k frames, windowed, then exit
make bench                   # headless Hopf --preset 4090, 600 dirty frames
make bench-windowed          # same, Wayland window, then exit
make bench-smoke             # tiny Hopf scene, no capture
make bench-gradient          # --scene gradient --preset 4090, 32×32 lattice
make bench-gradient-windowed # speakers + glass + 65k bed, then exit
make bench-gradient-record   # same look, encode mp4 via --record
make bench-hold              # two-clock skip: static once, pulse every 30
make bench-hold-record       # hold, in-sheet 1440p encode (no HUD)
make bench-loom-smoke        # inverse-Hopf loom, 60 frames, no capture
make bench-loom              # 4090 loom, 60 headless frames, no capture
make bench-loom-windowed     # 4090 loom, 900 frames, cinematic --record
make bench-core              # 64×64×64 ocean manifold, 60 headless
make bench-core-windowed     # 64×64×64, green frame, until Esc
```

`--frames N` exits after N presents (windowed or headless). `--frames 0` is
unlimited windowed (Esc still prints stats).

## Benchmark

Software fact of `qga-gpu-bench`. Not a proof of inner_cone mosaic or qga-app
cosmos. Hopf generator and skyrmion-ocean field are **Model** (`glam` in the
bench crate). The renderer stays an upload path.

Scenes:

| `--scene` | CLI default | What it is |
|-----------|-------------|------------|
| `hopf` / `sculpture` | yes | Hopf field + observer sculpture |
| `gradient` / `ngsm` | `make demo` | Lattice of instanced orbs + one thin ring per cell. **Model**: ngsm “gradient / structure”; rainbow ocean and ring tilts from a Stokes-skyrmion-like field (Chen et al. 2026; Kinder arXiv:2607.16520). Not a p5.js port and not those papers’ data. `--fluid` adds the 65 536-particle bed. |
| `hold` | `make bench-hold` | Frozen fiber lattice + two cones + separator torus. **Model**. Static topology once; 8–16 live tubes and 16k motes pulse every 30 frames; `aperture` / `height_scale` / `zener` / `time` breathe the rest. Camera sits in the sheet (not a bird’s-eye). `make bench-hold-record` is grid 64, 1440p, no HUD. Shows the hash-skip path the 65k ocean never prints. |
| `loom` / `braid` / `fabric` | `make bench-loom` | Cartesian N×N warp/weft (static) + cells on three S² latitudes inverse-Hopf to nested tori (live) + particle fill. **Model**: sculpt the chart by which cells are on and (θ,φ,ψ); do not upload 2D squiggles. `--flux elliptic` (default), `--lambda`, `--mosaic`. `--dirty-fibers` grows needles first. Not a fabricated silica loom. |
| `core` / `coreloom` / `volume` | `make bench-core` | 64×64×64 packed lattice. **Model**: ocean manifold (displacement \(+\mathbf{k}\), colour gradient \(-\mathbf{k}\)); inner alpha \(>0.5\), outer \(<0.5\). Class I icosahedral geodesic polyhedra (\(T=n^2\), \(F=20n^2\), default 2v → 80 faces) bounce as RGB shells. Green `#00FF00` frame follows grid resolution, not orb density. |

`make demo` is the public 65k scene. `make ring` is the 4k dirty-particle
smoke. Hopf bench is a different scale: observer sphere + cyan/orange cones +
gold torus (tessellated once), live Hopf tubes via `write_live_fibers`, flux
motes via `write_particles`, instanced `draw_geodesic_orb`. Live params stay
uniforms.

```bash
make bench                    # headless hopf 4090, 600 dirty frames, first+last capture
make bench-windowed
make bench-smoke              # 256 fibers / 4k motes / 60 frames, no capture
make bench-gradient           # 32×32 lattice, dirty rings
make bench-gradient-windowed  # --fluid 65k bed
make bench-gradient-record    # --grid 64 --fluid → mp4 (capture Wait; not a ring proof)
make bench-hold               # headless 300, --no-capture; static once, pulse skip
make bench-hold-record        # --grid 64 1440p in-sheet → mp4 (capture Wait; not a skip proof)
make bench-loom-smoke         # warp=16 elliptic, 60 frames, no capture
make bench-loom               # 4090 loom, 60 headless, no capture
make bench-loom-windowed      # 4090, 900 frames, cinematic --record
make bench-core               # 64×64×64 ocean manifold, 60 headless
make bench-core-windowed      # same volume, until Esc
./benchmarks/run.sh           # smoke then hopf 4090 --no-capture; JSON under benchmarks/results/
```

Hopf preset `4090` (this machine): 4096 fibers × 64 samples, 262 144 particles,
1024 orbs, glow, 1920×1080, 600 frames. Caps refuse before wgpu OOM
(`--particles` ≤ 8 388 608, combined 32 B records ≤ 1 GiB).

Acceptance (headless): `static_uploads == 1`; dirty particles
`particle_skipped == 0` and `ring_copies + particle_fallbacks >= frames`;
dirty fibers/rings do not no-op every frame; `particle_grows == 0` after
warmup on `4090` / `ring-qga`. Fallbacks counted, not fatal. Do not add a 4th
ring slot unless windowed fallbacks exceed ~1%.

`make bench-hold` is the skip-path proof the ocean never prints. This 4090
(300 headless, `--no-capture`): `static_uploads=1`, `live_fiber_writes=10`,
`particle_skipped=291`, `ring_copies=10`, `particle_fallbacks=0`.
`make bench-hold-record` is the same look, in-sheet, 1440p, HUD off
(capture Wait; not a skip proof).

## Consumers

Software fact. Pin `b9c9994`. Engine visual tip `7e7866b`. No `v0.1.0`.

| Consumer | Dep | Not in this extract |
|----------|-----|---------------------|
| stranger / other crate | `qga-gpu = { git = "…/qga_gpu", rev = "b9c9994…" }` | pin that sha; do not float `main`; no tag yet |
| [`inner_cone`](https://github.com/kinaar8340/inner_cone) @ `89a890c` | git, `qga_engine@7e7866b` + `qga_gpu@b9c9994`, `features = ["capture"]` | git consumer, not a path dep; does not print `UploadStats` |
| [`qga_engine`](https://github.com/qga-lab/qga_engine) @ `7e7866b` | git, `rev = "b9c9994"`; `features = ["winit", "headless", "capture", "glow"]` | do not PR a pin bump from this extract |

`capture` is the right feature set for `inner_cone --export`. Record layout
changes here cannot silently land in published consumers: they pin this sha,
not `main`. `Cargo.lock` is not the pin.

Engine scenes (lab / realm / cosmos / oam / reveal), CLI, and controls live
in that repo’s README and `docs/SCENES.md`. Software fact of `7e7866b`:
cosmos default 262 144 bodies (cap 524 288), realm 128 × 128 fibers and
256² terrain. Space on cosmos hides HUD tabs; it does not pause. This crate
does not implement those scenes.

## Dirty-flag / instance / persistent-particle policy

Software fact. This is the inner_cone upload contract, implemented in the
renderer so callers do not rebuild GPU topology every frame.

**Dirty flag (static topology).** `retain_meshes` / `retain_static_fibers`
hash mesh kind + lod + tube radius + centerline bytes. Identical re-upload is
a no-op. `UploadStats.static_uploads` counts real GPU writes; `make headless`
requires `static_uploads == 1` across 8 frames, `make demo-headless` the same
across 30 frames of the 65k ocean. `static_skipped` counts
the hits. `mark_static_dirty` forces the next retain. Live params
(`aperture`, `height_scale`, `zener`, `time`) are **frame uniforms** — they do
not retessellate cones/spheres/tori.

**Instance.** Repeated geodesic orbs use `draw_geodesic_orb(transform, color, lod)`
against a unit sphere retained at `Renderer::new`. One mesh, N instances
(`GpuOrbInstance`, 32 bytes: offset, scale, color, lod).

**Persistent particles.** `write_particles` keeps a GPU vertex buffer and a
3-slot `MAP_WRITE` staging ring (128 KiB slots, grow ×2). Hash skip
(`particle_skipped`) is the cheapest path. When bytes change: pick **any**
ready slot, memcpy, queue a `copy_buffer_to_buffer` on the next `render`.
`Queue::write_buffer` only if **zero** slots are ready (`particle_fallbacks`);
outstanding `pending` copies are never dropped. `particle_grows` is counted
apart from `fiber_reallocs`. Reclaim maps the slot cap (tight pow2, not a fat
arena). Profile `ring_copies` vs `particle_fallbacks` on this 4090 before
meshlets. Dirty-every-frame smoke:

```bash
make demo-headless   # 30 frames, 65k ocean, --no-capture
make ring            # 300 frames headless 4k; capture first+last only
make ring-windowed   # 300 presents; mailbox/FIFO in-flight
cargo run -p qga-gpu-demo --release -- --headless --frames 300 --dirty-particles
cargo run -p qga-gpu-demo --release -- --dirty-particles --frames 300
```

Acceptance (dirty): `ring_copies + particle_fallbacks >= frames`,
`particle_skipped == 0`, `static_uploads == 1`. Fallbacks are **allowed
and counted**. This 4090 (`f263ea7`): headless 8 still → `static_uploads=1`,
`ring_copies=1`, `particle_skipped=9`, `write_buffer=15` (uniforms / small
pending-writes, not mesh rebuilds). Headless 300 dirty first+last-capture →
`ring_copies=301`, `particle_fallbacks=0`, `particle_grows=0`. The extra copy
is init or the first captured frame. Zero fallbacks means the 3-slot ring
reclaimed before the CPU lapped `map_async`. Do **not** add a 4th slot on
this result (only if windowed fallbacks exceed ~1%). Windowed FIFO 300 → 0
fallbacks (present paces reclaim). That is DMA into DEVICE_LOCAL, not a
broken ring. These numbers do **not** prove `inner_cone` mosaic / hull /
live harmonics or `qga-app` lab / realm / cosmos / oam / reveal. Do not persist-map the particle VB through wgpu. Do not
`poll(Wait)` on write maps (`map_async` latency is 1–3 GPU frames;
Wait serializes them). Do not put particles on `StagingBelt`.
A 16 KiB belt is only for a future HUD/hub storm of tens of small
copies per present.
Frame loop is one encoder + one `submit` (pending-writes then the pass).
Empty `submit([])` is for asset load, not the present loop.

**Live vs static fibers.** Static separator / torus / held lattice:
`retain_static_fibers`. Live harmonics: `write_live_fibers` (same hash+radius
no-op). `UploadStats.live_fiber_writes` counts real live GPU writes, apart
from `ring_copies` and `static_uploads`. Tubes are centerlines + shader
extrusion, not CPU ribbons. `make bench-hold` is the split: static once,
live hash changes every 30 frames.

## Public surface (v0)

```rust
Renderer::new
Renderer::retain_static_fibers
Renderer::write_live_fibers      // no-op if topology hash + tube_radius unchanged
Renderer::retain_meshes          // sphere/cone/torus tessellated once
Renderer::draw_geodesic_orb(transform, color, lod)
Renderer::draw_geodesic_orb_alpha(transform, color, alpha)
Renderer::upload_hubs
Renderer::write_particles        // staging ring; skip if bytes unchanged
Renderer::write_hud
Renderer::render(gpu, cam, vis, t, capture)
```

## Features (`qga-gpu`)

| Feature | Default | What it does |
|---------|---------|----------------|
| `winit` | yes | `GpuContext::init_windowed` + pollster |
| `headless` | no | pollster; `init_headless` |
| `capture` | no | BGRA readback |
| `glow` | no | 9-tap bloom + Reinhard |

## Records

Software fact:

- Frame uniforms are **256 bytes**.
- Particle, fiber-point, hub, and orb-instance records are **32 bytes**.
- Buffer copies/maps: 4 B / 8 B. Texture `bytes_per_row`: **256 B** (capture).
  That pitch is WebGPU-on-Vulkan, not the 4090’s optimal tiling.
- wgpu validation is an enum tree (`BufferAccessError`, `TransferError`), not
  `VkResult`. Native default is a panic with a `Caused by:` tree — debug the
  description + `In …::method` frame, not a numeric code. Failures cluster on
  map 8, copy 4, texture pitch 256, and submit-while-mapped. The `source`
  chain is ContextError → wgpu-core; HAL is Internal/OOM. `layout.rs` keeps
  illegal numbers off the upload path. `particle_fallbacks` is not a
  validation error. No per-frame error scopes; uncaptured validation panics.
  Do not add wgpu-core just to downcast. wgpu trace is a WebGPU API log
  (RON + `player`); RenderDoc is a Vulkan frame capture — opposite sides
  of wgpu-hal. wgpu 24 cannot record traces. Validation → trace (when the
  dep allows); picture/barrier → RenderDoc on windowed 4090; counters
  stay in-process. Neither on default `make ring`. On this box wgpu is
  `ash` / `wgpu-hal::vulkan`: one graphics queue (family 0), HOST_VISIBLE
  ring, DEVICE_LOCAL VB. Do not call HAL; do not skip `map_async`.
  gpu-alloc is HAL `VkDeviceMemory` suballoc (wgpu 24: zakarumych/gpu-alloc),
  not the ring. Do not depend on it. `particle_grows=0` on `make ring` is
  the HAL steady state.
- Tubes are centerlines + shader extrusion, not CPU ribbons.

## Related repos

| Repo | Role |
|------|------|
| [`qga_engine`](https://github.com/qga-lab/qga_engine) | Scenes, math, sim. Pin this crate by git rev (`b9c9994`), never `main`. No `v0.1.0`. |
| [`qga`](https://github.com/qga-lab/qga) | Manuscript + pedagogical Python |
| [`inner_cone`](https://github.com/kinaar8340/inner_cone) | Sculpture viewer (Model). Git-pins `qga_engine@7e7866b` + this sha. |

## What this crate is not

- Realm terrain, cosmos n-body, OAM PDE, or reveal Lorenz scenes.
- A default `qga-math` / `qga-sim` dependency.
- A theorem about the Z-map or 350/π.

See [DESIGN.md](DESIGN.md) and [MIGRATION.md](MIGRATION.md).

## License

Geometry libraries are MIT. Several VQC repos are PolyForm Noncommercial plus patent notice US 63/913,110. This repo is MIT.
