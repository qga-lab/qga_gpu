# qga-gpu — design

A standalone wgpu/Vulkan renderer. **This crate owns the frame.** Geometry
meaning lives in [`qga_engine`](https://github.com/qga-lab/qga_engine)
(`qga-math` / `qga-sim` / `qga-app`). Renderer claims are **Software fact**.

This is not the QGA engine, not a fantasy realm, and not a solar-nebula sim.
Those scenes stay in [`qga_engine`](https://github.com/qga-lab/qga_engine)
(`main` @ `cd5081a`) and `inner_cone`. This crate is the upload path and the
swapchain.

## Crate map

```
qga_gpu/
├── crates/qga-gpu/                 # library
│   ├── src/{context,camera,renderer,types,mesh,hud,profile}.rs
│   └── src/shaders/{fiber,particle,hub,face,line,hud,blit,post}.wgsl
├── crates/qga-gpu-demo/            # 4k sculpture smoke
└── crates/qga-gpu-bench/           # public 65k ocean + Hopf / loom / hold / core-loom bench
```

| Crate | Role |
|-------|------|
| `qga-gpu` | Vulkan device, pipelines, resident buffers, upload API |
| `qga-gpu-demo` | 1 sphere, 2 cones, separator torus, 4k particles (`make demo-tiny`) |
| `qga-gpu-bench` | Public demo: 65k particle ocean (`make demo`). Hopf bench is `--scene hopf`. `hold` is the two-clock skip (`make bench-hold`). `loom` is inverse Hopf from a Cartesian chart, latitudes → nested tori (`make bench-loom`). `core` is coincident-addressed stacked lattice (`make bench-core`): X/Y half-select, Z inhibit, discrete \(N_\mathrm{sk}\). Geometry is glam **Model**. |

WGSL lives in-tree under `crates/qga-gpu/src/shaders/`. No runtime Python.

## Sources of truth

| Concern | Upstream | Here |
|---------|----------|------|
| Quaternions, Hopf, Hurwitz, topographs | `qga` / `qga-math` | not imported (optional `qga-math` feature) |
| Tube look, bloom, void | `flux_hopf_explorer` via engine WGSL | `src/shaders` |
| Static lattice once, live harmonics per frame | `inner_cone` engine tick | `Renderer` upload API |
| Realm / cosmos / OAM / reveal | [`qga_engine`](https://github.com/qga-lab/qga_engine) `qga-app` (`docs/SCENES.md`) | **not copied** |

Python is an authoring language in the ecosystem. The frame loop is Rust +
Vulkan. Do not `sys.path` into sibling repos.

## Why Vulkan / wgpu

This box is Wayland + NVIDIA 580 + Vulkan 1.4 ICD. `wgpu` selects the Vulkan
backend, talks Wayland WSI through `winit`, and keeps raster on the 4090
queue. CUDA is the wrong home for the swapchain.

FIFO present by default. Mailbox is detected (`present_mailbox`) but not used:
driver 580 has presented empty frames with mailbox. Software fact.

## Runtime

```
qga-gpu-demo  |  inner_cone  |  qga-app
    │
    └── qga-gpu
            ├── GpuContext  Vulkan device, optional surface, depth
            ├── Camera      orbit / fly, Y-up look_at_rh
            └── Renderer
                    ├── retain_meshes: sphere / cone / torus once
                    ├── static fiber pass vs live harmonic pass
                    ├── geodesic orb instances
                    └── post: blit, or glow (threshold + 9-tap + Reinhard)
```

Draw what was uploaded. Live params (aperture, height_scale, zener, time) are
frame uniforms.

Frame uniforms are 256-byte aligned. Particle, fiber, hub, and orb-instance
records are 32 bytes.

## Buffer policy

| Buffer | Lifetime | CPU write |
|--------|----------|-----------|
| Static fibers (centerline storage) | Resident. Hash(kind, points, tube_radius). | `retain_static_fibers` / torus in `retain_meshes`. No-op if hash matches. Counted in `static_uploads`. |
| Live fibers | Resident, grow ×2. | `write_live_fibers` only if hash or radius changed. Counted in `live_fiber_writes`. |
| Faces / lines / hubs / HUD | Resident, grow ×2. | On retain / explicit upload. |
| Geodesic orb mesh | Tessellated once in `Renderer::new`. | Instances via `draw_geodesic_orb`. |
| Particles | Persistent GPU VB + 3×128 KiB `MAP_WRITE` ring (grow ×2). | Skip if bytes unchanged (`particle_skipped`). Any ready slot; `write_buffer` only if none ready (`particle_fallbacks`). Never drop `pending`. Grows counted in `particle_grows`, not `fiber_reallocs`. |

Tubes are shader-extruded from 32-byte `GpuFiberPoint` records. Do not add
meshlets because of particle fallbacks.

## Mapping on a discrete 4090 (Software fact)

The particle VB is `VERTEX | COPY_DST` (DEVICE_LOCAL). The ring is three
128 KiB `MAP_WRITE | COPY_SRC` buffers (HOST_VISIBLE + HOST_COHERENT staging).
CPU writes the map forward-only, unmaps, GPU `copy_buffer_to_buffer` into VRAM.
A slot is either mapped on the CPU **or** in a submitted copy, never both.

wgpu will not give a persistent CPU pointer to that VERTEX buffer without
`MAPPABLE_PRIMARY_BUFFERS`. Do not put the 4k field in BAR/ReBAR to skip the
copy: vertex fetch from write-combine mapped VRAM is the wrong trade. Do not
emulate `vkMapMemory` and leave it mapped — wgpu forbids using a `MAP_WRITE`
buffer in a copy while it stays mapped. Compute-update of particles is a v0
non-goal.

`Queue::write_buffer` stays the fallback when zero slots are ready (native
wgpu often allocates a short-lived staging chunk per call). Measured on this
box, dirty 4k particles:

| Harness | `ring_copies` | `particle_fallbacks` | `particle_grows` |
|---------|---------------|----------------------|------------------|
| Headless 8 still, capture first+last | 1 | 0 | 0 |
| Headless 300 dirty, capture first+last | 301 | 0 | 0 |
| Windowed FIFO 300 dirty | 301 | 0 | 0 |

8 still (`f263ea7` on this 4090): `static_uploads=1` (topology once).
`particle_skipped=9` with `ring_copies=1` is hash-skip after the first field
lands. `write_buffer=15` is uniforms / small pending-writes, not mesh rebuilds.

300 dirty: `301 + 0 >= 300`, `particle_skipped=0`. The extra `ring_copies` is
init or the first captured frame. Zero fallbacks means the 3-slot ring
reclaimed before the CPU lapped `map_async`. Do **not** add a fourth slot on
this result. A 1-fallback first+last run is still allowed (CPU ahead of
`map_async`); it is not the operating point measured here.

Full capture `Wait` every frame idles the GPU before the next write → 0
fallbacks (hides pressure). FIFO present gates the CPU → reclaim always wins.
Do not `poll(Wait)` in the particle ring. Grow Waits, then rebuilds all three
slots; `particle_grows=0` at 4k is the HAL steady state.

Acceptance: dirty writes land as
`ring_copies + particle_fallbacks >= frames`. Fallbacks are allowed and
counted. `static_uploads == 1`. `particle_skipped == 0` when dirty.
DMA into DEVICE_LOCAL; map only HOST_VISIBLE staging; never wait the
swapchain on capture.

`UploadStats.static_uploads` is also the static-topology counter: default
headless 8-frame run must print `static_uploads=1`. These demo numbers do
**not** prove `inner_cone` mosaic / hull / live harmonics or `qga-app`
lab / realm / cosmos / oam / reveal. Those binaries still do not print `UploadStats`.

### `map_async` cost (Software fact)

`map_async` is cheap to **call** and expensive to **wait for**. The callback
cannot fire until every submitted use of that buffer is done **and** something
polls (`queue.submit`, `device.poll`, `instance.poll_all`). Native wgpu is
callback-based; a `Future` wrapper only hides a `poll`. Headless `Poll` in a
tight loop historically did **not** retire maps; `Wait` did. Windowed present
+ `submit` usually drains callbacks.

```
unmap slot
encoder.copy  staging → VB
submit
map_async(Write, 0..cap)   // AtomicBool in cb
write_particles: poll(Poll); test ready[]
```

That Poll is optional bookkeeping. It does **not** guarantee the previous
frame’s map finished. Fallback = all three still waiting.

| Piece | Cost |
|-------|------|
| `map_async` call | CPU µs |
| Wait for last copy of that slot | **1–3 frames of GPU work** (the real latency) |
| `vkMapMemory` / invalidate at 128 KiB | small; scales with **mapped range** |
| memcpy 128 KiB | bandwidth; CPU `Vec` already exists |
| `unmap` | flush if non-coherent; NVIDIA staging usually coherent |
| `device.poll(Wait)` | **stalls the GPU timeline to idle** — capture only |
| Mapping 2 MiB / 128 MB “just in case” | fat-map tax (Chrome 4090: 1.3–2× vs used range) |

Native Vulkan fat-map tax is smaller than in-browser, but `slice(..)` still
marks the whole cap live. Payload-sized slots keep that honest.

A first+last-capture fallback, when it happens, is **map-async latency**,
not memcpy. This 4090 sat at 0/300: reclaim beat the CPU. `poll(Wait)` after
every `map_async` would also zero fallbacks and destroy overlap
(serial-await). Hash skip still beats any map.

Poll policy:

- Frame loop: `submit` only. Windowed event loop already pumps wgpu.
- `write_particles`: `poll(Poll)` optional, for fresher `ready[]` same call.
- Capture read: `map_async(Read)` + `Wait` **only** on grabbed frames.
- Grow: `Wait` before destroying mapped slots.

Do not add async/await wrappers, extra poll threads, or
`on_submitted_work_done` per particle write. `particle_fallbacks` under dirty
windowed vs first+last headless **is** the map-async overhead meter. 0 per
300 frames at 128 KiB is the operating point on this 4090; 1 is still
healthy. Do not add a 4th slot unless windowed fallbacks exceed ~1%.
Do not map the VERTEX buffer.

## StagingBelt (Software fact — do not add for particles)

`wgpu::util::StagingBelt` is wgpu’s arena over the same MAP_WRITE + COPY_SRC
ring this crate built by hand. It wins when **one submit contains many small
copies**. It does not beat a dedicated 128 KiB particle slot.

Belt chunks: `active` (mapped, bump this frame) → `finish` unmaps → submit →
`recall` `map_async` → `free`. `Queue::write_buffer` still often allocates a
short-lived chunk **per call**. The belt amortizes map/unmap across N writes
on the **same** encoder. Chunk size must be larger than the biggest write and
about ¼–1× bytes per `finish()`. A write that will not fit allocates a
one-off chunk (`size.max(chunk_size)`).

| Upload | Size / cadence | Belt? |
|--------|----------------|-------|
| Particles (dirty) | one 128 KiB blit / frame | **No.** 3-slot ring is the belt with `chunk_size = payload`. |
| Frame uniforms | 256 B / frame | **No.** One `write_buffer` cheaper than finish/recall. |
| Static fibers / meshes | once (`static_uploads=1`) | **No.** |
| Live fibers | rare (hash skip) | Later, only if many short centerlines per tick. |
| Hubs / HUD / geo instances | packed `Vec` + one `write_grow` | **Yes, if** tens of copies per present. Not today. |

Dirty 4090 runs: `ring_copies≈frames` (301 on 300 dirty), `particle_fallbacks=0`,
`write_buffer≈1/frame` (uniforms; 307 on 300 dirty). A particle belt would
`map_async` the **whole** chunk — the fat-map trap if `chunk_size` ≫ 128 KiB.

| | 3-slot particle ring | StagingBelt |
|--|----------------------|-------------|
| Allocation | fixed 3 × payload | bump inside `chunk_size`, else new buffer |
| Close | unmap one slot | unmap **every** active chunk, even 1% full |
| Reclaim | `map_async` that slot | `map_async(slice(..))` **whole chunk** |
| Fallback | `write_buffer` if 0 ready | create another chunk (silent growth) |
| Best write | one 128 KiB field | dozens of 16–256 B scraps |

“Fast” belt: one `finish()` per submit, 2–3 chunks in flight, non-blocking
`recall`, DEST DEVICE_LOCAL, **many** writes sharing one mapped chunk. One
belt write per submit is `write_buffer` with extra ceremony. `chunk_size` =
256 KiB “so everything fits” is the unused-range tax. Do not `submit([])`
every N belt writes unless streaming assets. Do not map, write 256 B,
`finish`, repeat 300 times.

If a belt is added: `finish` → `submit` → `recall` (or
`finish_and_recall_on_submit` then submit **that** encoder before the next
allocate). Count `belt_chunks_created`; it must plateau by frame ~3.

If inner_cone later storms HUD/hubs (`write_buffer_calls` tens per present):
add a **16–64 KiB** belt on that path only, dest still DEVICE_LOCAL
`VERTEX|COPY_DST`. Count `belt_writes` / `belt_chunks_created` separately from
`ring_copies` / `particle_fallbacks`. Do not enable
`MAPPABLE_PRIMARY_BUFFERS`. Do not `Wait` on `recall`. Cap ~3 chunks in
flight, same as 3 slots. One encoder per frame.

Belt `allocate`: (1) first **active** chunk with room after
`align_to(offset, max(user_align, MAP_ALIGNMENT))`; (2) drain `map_async`
into **free**; (3) first free chunk that fits; (4) else
`create_buffer(size.max(chunk_size), MAP_WRITE|COPY_SRC, mapped_at_creation)`.
`finish` unmaps every active chunk **even if 1% full**. `recall` remaps
**full range**. Offset resets only when a chunk returns to free.

| Symptom | Cause |
|---------|--------|
| Chunk count grows every frame | `recall` after a submit that never happens, or CPU ahead of `map_async` (same as `particle_fallbacks`) |
| Many chunks, barely used | `chunk_size` ≫ per-`finish` bytes, or one write > remaining space |
| One huge chunk per write | single write > `chunk_size` → one-off `size`, never reuses well |
| Map cost scales with cap not payload | `recall` maps the whole chunk |

Steady state after warmup: **created ≈ 2–3**, `free+active+closed` bounded,
`bytes_written / capacity` not ≪ 1. On `make ring-windowed`, if
`belt_chunks_created` (when a belt exists) climbs past ~4, `recall` is late
or `chunk_size` is smaller than the largest write.

| Per-submit payload | `chunk_size` | After 300 frames |
|--------------------|--------------|------------------|
| 256 B uniforms only | 16 KiB | 2–3 resident, ~1.5% full — waste, still cheap |
| uniforms + HUD + hubs (~2–8 KiB) | 16 KiB | good |
| + one 128 KiB particle blit | ≥128 KiB | particles steal the belt; the ring is better |
| 128 KiB particles + 256 B | 256 KiB | fat map; worse than a tight slot |

## Queue submit (Software fact — pattern A)

Two streams meet at `submit`: wgpu **pending-writes** (`Queue::write_buffer`
memcpy into impl staging, GPU copy flushed **at the start of the next
submit**), then your `CommandBuffer`s. `submit([])` flushes pending-writes.
Staging is released after **that** submit retires. Many `write_buffer`s with
no submit = many live impl stagings (CubeCL flushes every 64). This crate
does ~1 `write_buffer`/frame plus rare mesh — no empty-submit in the present
loop.

```
write_particles            // map slot or fallback write_buffer (pending-writes)
write_buffer(uniforms)     // pending-writes
encoder:
  copy staging → particle VB
  render / blit / optional capture
submit([encoder])          // pending-writes first, then encoder
map_async particle slot
present
```

That is **one encoder, one submit**. Particle copies live on the encoder
(`ring_copies`). Uniforms live on pending-writes (`write_buffer_calls`). A
belt would move small copies onto the encoder and add `finish`/`recall`; it
would not change submit count.

Do **not**: upload-then-draw double `submit` (128 KiB + 256 B is not a fat
stream); `submit([])` mid-present; submit while a buffer is mapped (ring
unmaps before `pending`; belt `finish` unmaps first; capture maps **after**
submit, first+last only); a transfer queue (wgpu v0 is one graphics queue —
overlap is CPU record vs GPU previous frame).

Keep DEST DEVICE_LOCAL. One graphics submit per present until
`write_buffer_calls` per frame is tens, or you stream megabytes before the
pass.

## Alignment and texture staging (Software fact)

Portable wgpu table (WebGPU + D3D12), not a 4090 quirk. Copy/map and bind
are different problems. Texture row pitch is **not** a buffer-copy rule.
`layout.rs` pins the subset this crate uses plus the full constant values.

| Symbol | Value | Must be multiple of |
|--------|-------|---------------------|
| `COPY_BUFFER_ALIGNMENT` | **4** | `copy_buffer_to_buffer` / `clear_buffer` offset **and size**; `mapped_at_creation` buffer size |
| `MAP_ALIGNMENT` | **8** | `map_async` / `get_mapped_range` offset **and size** |
| `VERTEX_STRIDE_ALIGNMENT` | **4** | vertex offset and array stride |
| Immediate data (WebGPU) | **4** | `set_immediates` ranges (wgpu 24 has no named const) |
| Storage bind size | **4** | bound SSBO size (wgpu 24 has no named const) |
| Uniform bind offset | **256** | dynamic uniform offset; this crate’s UBO is **256 B** |
| `QUERY_RESOLVE_BUFFER_ALIGNMENT` | **256** | `resolve_query_set` dest offset |
| `QUERY_SIZE` | **8** | one query slot |
| `COPY_BYTES_PER_ROW_ALIGNMENT` | **256** | `copy_*_texture` **bytes_per_row only** — not `Queue::write_buffer` |

`Queue::write_buffer` dest offset + size need **copy 4**, not map 8, unless
you also map that dest. Belt: allocate size % 4; bump
`max(user, MAP_ALIGNMENT)`. `create_buffer_init` pads content length to ≥4
then copies the unpadded prefix.

A range that is **mapped and then copied** must be % 8 (implies % 4):
`0..8`, `0..32`, `8..40`, `n×32`. Illegal map: `0..4`, `0..12`. Illegal
copy: offset 2, size 2 or 6. `need = 0` is not a legal map — skip (empty
particles). Mappable staging **allocation** should be % 8 if you
`map_async(0..cap)`. Grow ×2 from 128 KiB stays legal.
`mapped_at_creation` alone only needs % 4.

`create_buffer_init` rounds content length up to a multiple of 4. StagingBelt
asserts copy size/offset % 4 == 0 and map align ≥ 8.

| Record | Size | Copy 4 | Map 8 | Bind |
|--------|------|--------|-------|------|
| `GpuParticle` / fiber / hub / orb | 32 | yes | yes | vertex/storage |
| `FiberMeta` | 16 | yes | yes | uniform |
| `FrameUniforms` | 256 | yes | yes | UBO 256 |
| `HudVert` | 24 | yes | yes | vertex |
| `FaceVert` | 48 | yes | yes | vertex |
| `LineVert` | 32 | yes | yes | vertex |
| `need = n×32` | | yes | yes | — |
| slot cap 128 KiB ×2 | | yes | yes | — |

This crate’s HUD vert is **24 B** (`[f32;2]` + `[f32;4]`), so `n×24` is
always % 8. A **12 B** HUD record would copy (`% 4`) but mapping `0..12` is
illegal; pad mapped/copy size to `align(n×12, 8)` or `write_buffer` without
mapping. Do not copy `particles.len()` without `× 32`. Dynamic uniform
offsets 256-aligned — one UBO at offset 0. Do not `resolve_query_set` into
the particle VB (dest offset % 256).

Uniform 256 B is a **bind** rule, not a copy rule: you can `write_buffer` 64 B
into a 256 B UBO; you cannot bind a 64 B std140 range on all backends.

## Texture pitch, origin, block size (Software fact)

Texture alignment is **block geometry + row pitch**. It is not
`COPY_BUFFER_ALIGNMENT` / `MAP_ALIGNMENT`. This crate only hits BGRA8
capture. Compressed and depth have extra rules we do not use yet.

| | `Queue::write_texture` | `copy_buffer_to_texture` / `copy_texture_to_buffer` |
|--|------------------------|-----------------------------------------------------|
| `bytes_per_row` % 256 | **not required** (impl pads) | **required** |
| Buffer `offset` | % block size (1 for RGBA8; 4 for depth/stencil aspect) | same |
| Origin x/y | % block width/height | same |
| Copy size w/h | % block, or flush to subresource edge | same |
| Sample count | 1 | 1 |
| Usage | `COPY_DST` | `COPY_DST` / `COPY_SRC` |

256 B row pitch is WebGPU/D3D12 portable. Vulkan would allow tighter; wgpu
will not on the **encoder** path. `write_texture` sets `aligned=false` and
row-copies into padded staging internally.

BGRA8 / RGBA8 block = 1×1 texel, 4 bytes:

```text
dense_bpr   = width * 4
padded_bpr  = ceil(dense_bpr / 256) * 256     // copy_*_texture only
buffer_size = padded_bpr * height              // 2D
```

| width | dense | padded |
|-------|-------|--------|
| 1920 | 7680 | 7680 |
| 1280 | 5120 | 5120 |
| 800 | 3200 | **3328** |

Last row in the spec may be dense; the buffer still reserves a full padded
stride per row. CPU pack walks `src += padded_bpr`, writes `width * 4`.
`rows_per_image` is required when depth/layers > 1; 2D capture uses height.
Capture offset 0 (Metal wants 16 on some buffer–texture copies; 0 is safe).
Do not start a capture plane at offset 4.

Origin ZERO, full `Extent3d { width, height, 1 }`. RGBA8 block 1×1: any
pixel origin is legal. Compressed (BC1/BC3): 4×4 blocks; origin and copy
size in texels % 4 except a copy that hits the mip edge; `bytes_per_row` =
blocks_across × block_bytes, then pad to 256 for encoder copies. wgpu docs:
32×16 RGBA8 encoder copy → dense 128, **padded 256**.

Depth/stencil: one aspect at a time; offset alignment **4**. MSAA
(`sample_count > 1`): buffer copies forbidden — resolve first. Do not reuse
`copy_bytes_per_row_bgra` for D32.

Capture: `copy_texture_to_buffer` with `bytes_per_row =
copy_bytes_per_row_bgra(width)`, origin 0, full extent, 4 B/pixel, samples 1.
Staging size = `capture_staging_bytes` = `padded_bpr * height`, MAP_READ.
Mapping that buffer is a **buffer** 8-byte rule; `3328 * h` is % 8.
`3200 * h` would be legal to **map** but illegal as `bytes_per_row`.
Color/bloom: GPU-only until capture (`COPY_SRC` if read back). CPU image
upload if ever: one shot → `write_texture` dense; streaming mips → pad 256
and `copy_buffer_to_texture`. Do not put particles in a texture.

Illegal: `bytes_per_row = 3200` on encoder copy; capture origin (1,0) on a
BC texture; `write_texture` of MSAA; mapping the capture buffer at offset 1.
BGRA8 full-frame origin 0 + padded pitch is the whole texture-alignment
surface this renderer has.

## wgpu “layout” vs Vulkan tiling and image layouts (Software fact)

“Texture layout” is three things. wgpu exposes **one**. Vulkan exposes all
three. On this 4090, wgpu-Vulkan already does the other two from usage flags
and implicit barriers.

| Meaning | Vulkan | wgpu / WebGPU |
|---------|--------|---------------|
| **Memory tiling** | `OPTIMAL` vs `LINEAR`; `vkGetImageSubresourceLayout` → `rowPitch` | Always opaque / optimal. No linear tiling, no GPU row-pitch query |
| **Usage layout** | `VkImageLayout`: `COLOR_ATTACHMENT_OPTIMAL`, `TRANSFER_SRC_OPTIMAL`, `SHADER_READ_ONLY`, `PRESENT_SRC_KHR`, `GENERAL`, `UNDEFINED`, … | Hidden. Usage bits + pass load/store. Impl inserts `VkImageMemoryBarrier`s |
| **CPU buffer for copies** | `VkBufferImageCopy`: `bufferRowLength` (texels; 0 = packed); pitch often 4 B / block, **not** a spec 256 | `TexelCopyBufferLayout`: `bytes_per_row` **% 256** on encoder copies |

This crate’s “layout” is only the third: capture `padded_bpr`. You never pick
tiling or `VkImageLayout`.

Vulkan **linear** images are row-major in device memory (map + `rowPitch` if
host-visible). Usually 2D, no depth, slow GPU access. **Optimal** images are
tiled/swizzled/DCC; CPU cannot interpret the bytes. wgpu textures are
optimal. There is no `TEXTURE_TILING_LINEAR` and no persistent map of a
color target. Readback is always `copy_texture_to_buffer` into a linear
**buffer**. That is why capture uses `padded_bpr`, not
`vkGetImageSubresourceLayout`. Swapchain present is `PRESENT_SRC_KHR` after
a transition wgpu owns.

Vulkan you write `UNDEFINED → TRANSFER_DST → copy → SHADER_READ_ONLY` or
`COLOR_ATTACHMENT → TRANSFER_SRC → copy to buffer → PRESENT`. Wrong
old/new layout is a race or a discard. wgpu you set usages at create.
Dawn/wgpu-core map a usage to a layout (`CopyDst` → `TRANSFER_DST_OPTIMAL`,
attachment → `COLOR_ATTACHMENT_OPTIMAL`; combined usages often `GENERAL`)
and barrier from the previous tracked use. “Cannot copy a texture that is
still a color attachment in this pass” means: end the pass first.
`VK_KHR_unified_image_layouts` is Vulkan catching up to stay in `GENERAL`.
Load/store (`Clear` / `Load` / `DontCare`) map to `loadOp`/`storeOp` and
whether the impl transitions from `UNDEFINED`. Scene pass `Clear`+`Store`
is `COLOR_ATTACHMENT_OPTIMAL`, then a barrier to `TRANSFER_SRC` if capture
runs.

Vulkan `bufferRowLength = width` (800 texels → 3200 B) is legal. The same
copy through wgpu **must** use 3328 B. `Queue::write_texture` is the escape:
dense 3200, impl pads into Vulkan-legal staging. `bufferOffset` 0 is the
portable choice (Metal 16 B on some paths).

| wgpu object | Vulkan analogue |
|-------------|-----------------|
| Offscreen color (`RENDER_ATTACHMENT \| COPY_SRC`) | optimal `VkImage`, color + transfer src |
| `begin_render_pass` | `COLOR_ATTACHMENT_OPTIMAL` |
| `copy_texture_to_buffer` | barrier → `TRANSFER_SRC_OPTIMAL`, `vkCmdCopyImageToBuffer` with 256-aligned row pitch |
| Capture buffer | host-visible **buffer**, not a linear image |
| Swapchain present | barrier → `PRESENT_SRC_KHR` |
| Particle VB | not an image; buffer copies, 4/8 |

Porting a Vulkan uploader with `bufferRowLength = width` fails wgpu until
you pad to 256. Porting wgpu capture to raw NVIDIA Vulkan can drop the 256
pad. Never `PREINITIALIZED` + linear tiling through wgpu. Do not expect
`GENERAL` vs `COLOR_ATTACHMENT_OPTIMAL` control for bloom vs capture —
wgpu transitions when the encoder records the copy after the pass.

Keep particles as DEVICE_LOCAL buffers. Keep color as optimal textures.
Treat 256 B pitch as a **WebGPU-on-Vulkan tax**, not the GPU’s true tiling.

## wgpu validation (Software fact — no VkResult table)

wgpu does not ship numeric error codes. Validation is a typed enum tree,
classified as WebGPU `ErrorType::Validation` (vs `OutOfMemory` / `Internal` /
`DeviceLost`). The panic text `wgpu error: Validation Error / Caused by:` is
`Display` on that enum. Native default: uncaptured validation **panics**.
`push_error_scope(ErrorFilter::Validation)` can catch; do not use it to retry
dense pitch.

`particle_fallbacks` is a **ready-slot miss**, not a validation miss.

**Buffer map** (`BufferAccessError`) — the ring: `UnalignedOffset` (`offset %
8`), `UnalignedRangeSize` / `UnalignedRange`, `AlreadyMapped`,
`MapAlreadyPending`, `NotMapped`, `MissingBufferUsage`, `OutOfBounds*`,
`NegativeRange`, `MapAborted`, `Failed`. `0..12` → unaligned. Empty `0..0`
must not map. Mapping a slot still used as copy source in an unsubmitted
encoder often fails on **submit**, not this enum.

**Buffer create** (`CreateBufferError`): `UnalignedSize` (`mapped_at_creation`
size % 4), `InvalidUsage`, `UsageMismatch` (`MAP_WRITE` + `VERTEX` without
`MAPPABLE_PRIMARY_BUFFERS`), `MaxBufferSize`, `AccessError`. Ring staging is
`MAP_WRITE | COPY_SRC` + `mapped_at_creation` + cap % 8. Dest VB is
`VERTEX | COPY_DST`, not mapped — that combo avoids `UsageMismatch`.

**Copies** (`TransferError`): `UnalignedBytesPerRow` (encoder texture copies,
256), `UnspecifiedBytesPerRow` / `UnspecifiedRowsPerImage`,
`InvalidBytesPerRow` (bpr < dense), `UnalignedBufferOffset`,
`UnalignedCopySize` (buffer–buffer % 4), `UnalignedCopyWidth`/`Height`/`Origin*`,
`MissingBufferUsage` / `MissingTextureUsage`, buffer overrun (`padded_bpr ×
rows` larger than allocation), `InvalidSampleCount` (MSAA), `InvalidMipLevel`,
`SameSourceDestinationBuffer`, `CopyAspectNotOne`. 800-wide BGRA at bpr 3200
→ `UnalignedBytesPerRow`. 3328 pitch with a 3200×height buffer → overrun.
`copy_bytes_per_row_bgra` + alloc `padded_bpr * height` keeps both silent.

Frame uniforms at 256 B dodge downlevel `BUFFER_BINDINGS_NOT_16_BYTE_ALIGNED`.

```text
write_particles map 0..need     → BufferAccessError::Unaligned* if need % 8 ≠ 0
copy staging → VB               → TransferError::UnalignedCopySize if need % 4 ≠ 0
copy_texture_to_buffer capture  → UnalignedBytesPerRow / bounds overrun
create MAP_WRITE|VERTEX         → CreateBufferError::UsageMismatch
submit while slot still mapped  → Validation on Queue::submit
```

`layout.rs` is the static version of these variants. It does not replace a
runtime scope; it keeps the happy path from constructing illegal numbers.
Grep strings (`UnalignedBytesPerRow`, `UnalignedOffset`) are `Display` text;
the **enum** is the API. There are no hex codes.

### `Error::Validation` source chain (Software fact)

A `Validation` is a wrapper. The useful part is the `source` chain: API call
context → wgpu-core enum → (sometimes) naga or a usage tracker. There is
**no** `VkResult` at the bottom; HAL failures are `Internal` or `OutOfMemory`.

```text
wgpu-core fn returns E: WebGpuError
  E.webgpu_error_type() == Validation
frontend: ContextError { fn_ident, source: E, label }
Error::Validation { source: Box<ContextError>, description: format_tree }
→ error scope if filter matches
→ else on_uncaptured_error / panic
```

`description` is the indented `Display` tree (`In CommandEncoder::copy_texture_to_buffer` / `Copy error` / `Bytes per row does not respect COPY_BYTES_PER_ROW_ALIGNMENT`). `source()` walks the same tree. `fn_ident` is the public method; `label` is the resource debug label if any.

| Layer | Type | What it checks |
|-------|------|----------------|
| Frontend | `ContextError` | which API call |
| Transfer | `TransferError` | copies, pitch, origin, usage bits |
| Buffer access | `BufferAccessError` | map range, 8/4, already-mapped |
| Buffer create | `CreateBufferError` | size % 4, usage combo, max size |
| Texture create | `CreateTextureError` | format, usage, samples, limits |
| Usage tracker | `Missing*Usage`, encode-scope | `COPY_SRC` vs `VERTEX`, submit-while-mapped |
| Init tracker | `MemoryInitFailure` | use-before-init |
| Bind / pipeline | `BindingError`, `Create*PipelineError` | layout vs shader |
| Shader stage | `validation::StageError` | naga I/O vs pipeline |
| Shader parse | naga | WGSL |
| Device | `DeviceError` | lost / OOM (**usually not** Validation) |
| HAL | `hal::DeviceError` | driver; Internal/OOM |

`wgpu_core::validation` is **shader interface** (bindings, varyings, workgroup), not copy pitch. Pitch lives in `transfer.rs`.

This crate can hit: ring `BufferAccessError::Unaligned*` / `MapAlreadyPending` / `MissingBufferUsage` (map dest VB); `CreateBufferError::UsageMismatch` / `UnalignedSize`; `TransferError::UnalignedCopySize` on ring copy; submit-while-mapped; capture `UnalignedBytesPerRow` / `InvalidBytesPerRow` / overrun / `MissingTextureUsage`. Draw: `StageError::Binding` if UBO 256 vs shader drifts; pipeline format mismatches. Shader modules fail at **create_shader_module**, not at draw.

Not a validation source: `map_async` `Err(BufferAsyncError)`; `particle_fallbacks`; `vkMapMemory` HAL fail; present-mode warn `Unrecognized present mode 1000361000`.

Do **not** add `wgpu-core` as a dependency to downcast `TransferError`. Tests grep `description` (e.g. `UnalignedBytesPerRow`). Debug dump:

```rust
fn dump(err: &wgpu::Error) {
    match err {
        wgpu::Error::Validation { description, source } => {
            eprintln!("{description}");
            let mut s: Option<&dyn std::error::Error> = Some(source.as_ref());
            while let Some(e) = s {
                eprintln!("  source: {e}");
                s = e.source();
            }
        }
        other => eprintln!("{other}"),
    }
}
```

Policy: validation sources are encode-time contracts. Lock numbers in
`layout.rs`; default uncaptured handler panics. A debug scope around capture
logs `description` and leaves. Do not branch on `TransferError` in
`write_particles`.

### Error scopes (Software fact — not used on the hot path)

Scopes are a LIFO **filter stack on the device**, per **OS thread**. They
capture the first matching `Error`. Anything that misses the stack hits
`on_uncaptured_error` (native default: panic). They do not replace
`layout.rs` or `particle_fallbacks`.

This crate is **wgpu 24**:

```rust
device.push_error_scope(ErrorFilter::Validation);
// create / encode / submit
let fut = device.pop_error_scope(); // pop is immediate; future is the result
gpu.device.poll(wgpu::Maintain::Wait);
let err: Option<Error> = pollster::block_on(fut);
```

Newer wgpu uses a `!Send` guard (`let scope = push...; scope.pop()`). Drop
without pop still pops and **discards** the error. Document the 24 API until
the dep moves.

`ErrorFilter`: `Validation` | `OutOfMemory` | `Internal`. A validation scope
does **not** swallow OOM. Nested: inner matching filter eats the error;
parent sees `None`. `Error`: `Validation { description, source }`,
`OutOfMemory { source }`, `Internal { description, source }`.

Pop only schedules the query. The future completes after the **device
timeline** has processed the enclosed commands — same pump as `map_async`.
`block_on(pop)` without `Wait` can hang or miss, same class as
`Maintain::Poll` vs capture maps. Native validation is often synchronous on
encode/submit; Web is async. Write as if async.

`on_uncaptured_error` may run **inline** on the producing thread. Do not take
locks the caller holds. Empty stack + default handler = panic
`wgpu error: Validation Error / Caused by:`.

Stack is OS-thread-local, not green-thread-local. `map_async` callbacks can
run on the poll thread. Do not push on the render thread and pop on a worker.

| Path | Scope? |
|------|--------|
| Particle ring map/copy | **No** — illegal numbers are tests; fallback is not a GPU error |
| Capture `copy_texture_to_buffer` | Optional debug-only around first encode |
| `create_buffer` grow | `OutOfMemory` only if a soft fail is wanted |
| Hot frame (`make ring`) | **No** — alloc + poll + future per frame is the wrong tax |
| `cargo test` of a known-bad pitch | Yes: push Validation, encode 3200 bpr, pop, assert `UnalignedBytesPerRow` |

Do not wrap `write_particles` in a scope and treat `UnalignedRange` as “use
`write_buffer`”. That hides a contract break. `map_async` callback
`Err(BufferAsyncError)` is **not** a scope event. Submit-while-mapped is
validation on `Queue::submit`. Capture `Wait` after read `map_async` is for
the map, not for popping a scope.

Policy: default uncaptured = **panic** (keep). No per-frame scopes. Tests stay
numeric (`need % 8`, `padded_bpr`). A scope is a diagnostic around one
experimental encode, popped with `Wait` on the same thread, never the
particle fallback path. Keep OOM on a **separate** outer scope if a grow can
legally fail.

`tests/layout.rs` pins both worlds: `size_of` % 4 and % 8; partial map
`particle_ring_need_bytes(n) = n × 32` for odd `n`; empty `n = 0` is 0 and
must not be mapped; grow ×2 from 128 KiB stays % 8; capture pitch
`copy_bytes_per_row_bgra` (1920/1280 exact, 800 → 3328). That is the portable
wgpu contract, not a 4090 quirk. Do not merge HOST_VISIBLE buffer staging
(4/8) with the capture MAP_READ buffer (`padded_bpr × height`). Do not put
the particle field in a texture to dodge alignment — that swaps a solved
32 B copy for a 256 B pitch path.

### Debugging a native validation panic (Software fact)

Native wgpu validation is a panic with a `Caused by:` tree unless you install
a handler or a scope. Debug the **description string + API frame**, not a
numeric code. This crate’s failures cluster on map **8**, copy **4**, texture
pitch **256**, and submit-while-mapped.

**Make the message complete.** Default panic already prints the tree. Keep
labels on buffers/textures so `ContextError.label` is not empty:
`part-stage-0` / `part-stage-1` / `part-stage-2`, `part-gpu`,
`capture-staging`, `scene-color` / `resolve-color`. `{e:#}` walks `source()`:

```rust
device.on_uncaptured_error(Box::new(|e| {
    log::error!("{e:#}");
    panic!("wgpu uncaptured: {e:#}");
}));
```

Do this once at device create, not per frame. A log-only handler **replaces**
the default panic — do not install that on `make ring`. Env that helps:

- `RUST_BACKTRACE=1` — Rust frame of *your* `copy_*` / `map_async`
- `RUST_LOG=wgpu_core=debug,wgpu_hal=warn` — tracker / map state (`map state -> Waiting`)
- `WGPU_TRACE=dir` — Firefox/Gecko RON dump, not this crate’s `request_device` (wgpu 24 `trace` is off; see below)

The `Unrecognized present mode 1000361000` line is **not** validation. Ignore it.

**Read the tree top-down.**

```text
wgpu error: Validation Error
Caused by:
  In CommandEncoder::copy_texture_to_buffer    ← fn_ident
    Copy error
      Bytes per row does not respect `COPY_BYTES_PER_ROW_ALIGNMENT`
```

| Top frame | Look at |
|-----------|---------|
| `Buffer::map_async` / `get_mapped_range` | `need % 8`, cap, already mapped |
| `Device::create_buffer` | usage combo, `mapped_at_creation` size % 4 |
| `CommandEncoder::copy_buffer_to_buffer` | offset/size % 4, overrun, same buffer |
| `copy_texture_to_buffer` / `to_texture` | `bytes_per_row % 256`, buffer len ≥ `bpr × h`, `COPY_SRC` on texture |
| `Queue::submit` | mapped buffer used in that encoder; destroyed resource |
| `Device::create_render_pipeline` | shader `StageError` / targets |
| `Device::create_bind_group` | min binding size vs 256 B UBO |

Inner variant names (`UnalignedBytesPerRow`, `UnalignedOffset`,
`UsageMismatch`) are the real “codes.”

**Isolate with a scope (debug only).** This crate is wgpu 24 (`push` /
`pop_error_scope` + `Maintain::Wait`, not the later `!Send` guard):

```rust
device.push_error_scope(wgpu::ErrorFilter::Validation);
encoder.copy_texture_to_buffer(/* ... */);
queue.submit([encoder.finish()]);
let fut = device.pop_error_scope(); // pop is immediate; future is the result
device.poll(wgpu::Maintain::Wait);
if let Some(e) = pollster::block_on(fut) {
    panic!("capture: {e:#}");
}
```

Same thread, pop immediately, `Wait` so the future resolves. Do not wrap
`write_particles` and treat the error as a `write_buffer` fallback.

**Checklist for this renderer.**

Ring:

- `need = n * 32`; `n == 0` → no map
- CPU write is `get_mapped_range_mut(0..need)` with `need % 8 == 0`
- reclaim `map_async(Write, 0..cap)` with `cap % 8 == 0` (128 KiB min, grow ×2)
- unmap before the encoder copies that slot
- dest VB: `VERTEX | COPY_DST` only
- after submit, one pending map per slot (`MapAlreadyPending` = double reclaim)

Capture:

- `bytes_per_row = copy_bytes_per_row_bgra(width)` (800 → 3328)
- staging size `padded_bpr * height`, not `width * 4 * height` when those differ
- color target has `COPY_SRC`; copy **after** the render pass ends
- origin `ZERO`, samples 1, aspect `All` for BGRA

Submit:

- no `get_mapped_range` live on a buffer named in the encoder
- capture read map happens **after** submit, then `Wait`

If the tree says overrun but pitch is 256-aligned, the **allocation** is
still dense. Pad the buffer, not just the field.

**Confirm without the GPU.** `layout.rs` already encodes the numbers. To
assert a *message* you need a device (`push Validation` → encode bad 3200 bpr
→ finish/submit → pop → `description` contains `UnalignedBytesPerRow`). Do
not add that to `make ring`. Keep it next to `capture_row_pitch_is_256` if
you want a runtime twin; tests stay numeric until then.

**What looks like validation but is not.**

| Symptom | Actual |
|---------|--------|
| `map_async` callback `Err` | GPU still using slot / aborted |
| `particle_fallbacks > 0` | no ready slot; legal |
| Hang on `block_on(pop_error_scope())` | missing `poll(Wait)` |
| Black capture, no panic | copy never recorded, or Wait packed the wrong rows |
| Pipeline compile fail at startup | naga / `StageError`, fix WGSL |

**Fast path when it panics on the 4090.**

1. Read `In …::method`.
2. If transfer: print `width`, `padded_bpr`, `staging.size()`, `need`, `cap`.
3. If map: print slot index, `ready[]`, pending queue length, mapped flag.
4. If submit: which buffers are still `get_mapped_range`’d.

Do not turn validation into a recovery path. The debug job is to make
`layout.rs` and the encoder agree so the uncaptured handler never fires on
`make ring` or `make ring-windowed`.

### wgpu API traces vs RenderDoc (Software fact)

wgpu trace is a **WebGPU API log**. RenderDoc is a **Vulkan (or D3D) frame
capture**. They sit on opposite sides of wgpu-hal. Use them for different
bugs.

```text
qga-gpu  write_particles / encoder / submit
    ↓
wgpu-core Actions          ← TRACE (RON + blobs)
    ↓
wgpu-hal Vulkan
    ↓
vkCmd* / images / pipelines  ← RENDERDOC
    ↓
4090
```

Trace never sees `VkImageLayout` or tiled color. RenderDoc never sees
`map_async` callbacks, `ErrorFilter`, or `particle_fallbacks`.

**This crate is wgpu 24.** `request_device` still takes a positional
`trace_path` (`context.rs` passes `None`). The frontend `trace` feature is
**commented out** pending [gfx-rs/wgpu#5974](https://github.com/gfx-rs/wgpu/issues/5974);
a `Some(path)` logs that tracing was removed and wgpu-core still gets
`None`. Do not add `--features trace` until the dep can record. Later wgpu
uses `DeviceDescriptor { trace: Trace::Directory(path), .. }` (`Trace::Off`
by default). `WGPU_TRACE=/path` is Gecko, not this `request_device`.

| | wgpu `trace.ron` + data | RenderDoc `.rdc` |
|---|-------------------------|------------------|
| Unit | device API call (`CreateBuffer`, `WriteBuffer`, `Submit`) | GPU work in one present (or a range) |
| Buffers | sizes, usages, **CPU payloads** as files | device-visible contents at event |
| Textures | create + copy layout (`bytes_per_row`) | tiled/optimal image, sampled views |
| Shaders | WGSL recorded as data | SPIR-V that actually ran |
| Sync | submit / map actions as text | barriers, layout transitions wgpu inserted |
| Time | lockstep, no clocks | timestamps, duration per event |
| Surface | swapchain actions if any | present; Firefox traces often **no** swapchain |
| Size | small API + every upload blob | large; one 1920×1080 frame + resources |

| Symptom | Tool |
|---------|------|
| `UnalignedBytesPerRow`, `UsageMismatch`, map `0..12` | **Trace** (and `layout.rs`) |
| Submit-while-mapped, missing `COPY_SRC` | Either; trace is enough |
| Capture nonempty but wrong pitch packing | Trace shows bpr; RenderDoc shows the image |
| Black frame, validation clean | **RenderDoc** — pass targets, viewport, draw counts |
| Bloom / glow look wrong | RenderDoc texture viewer |
| `particle_fallbacks`, ring slot rotation | **Neither** — CPU counters; trace only shows the copies that happened |
| “Is the tube extruded?” | RenderDoc mesh/VS output |
| File a wgpu issue on another machine | **Trace** + matching player revision |
| Nsight occupancy / PCIe | RenderDoc adjacent; not the RON player |

`gfx-rs/subscriber` (Chrome trace JSON) is archived CPU spans. Use
`RUST_LOG`. WebGPUReconstruct is browser WebGPU, not this native crate.

**How you capture on this box.**

Trace (once the dep restores it):

```text
mkdir -p /tmp/qga-wgpu-trace
DeviceDescriptor { trace: Trace::Directory(path), .. }   # feature "trace"
cargo run -p qga-gpu-demo --release --features trace -- --headless --frames 8 --dirty-particles
cd wgpu/player && cargo run -- /tmp/qga-wgpu-trace       # no winit: headless
# cargo run --features winit -- path                     # swapchain traces
```

Needs `wgpu` feature `trace`. Folder must exist. Crash: close the `]` in
`trace.ron` or the player dies on RON EOF. Replay is the same API stream
on a **matching wgpu revision**; the backend field in RON is editable
(`Vulkan` → `Dx12`) but brittle. Player is sequential, not a time-scrubber.
It will re-hit `UnalignedBytesPerRow`. It will not show NVIDIA tiling or
SGPR counts. You do not embed `Player` in qga-gpu.

You should see in RON, in order: create UBO, 3 ring slots, dest VB, color
target, capture buffer; per frame `WriteBuffer` uniforms,
`CopyBufferToBuffer` particles, render pass, maybe `CopyTextureToBuffer`
on frame 0 and last; `Submit`; `MapAsync`.

RenderDoc: launch `qga-gpu-demo` windowed (`mailbox=true`, 1280×720).
Headless has no present; RenderDoc can still inject if you capture after
`submit`, but the easy path is `make ring-windowed`. You will see:

- `vkCmdCopyBuffer` = ring slot → VB
- `vkCmdDraw` / indexed draws for fibers
- `vkCmdCopyImageToBuffer` = capture
- implicit `VkImageMemoryBarrier` COLOR_ATTACHMENT → TRANSFER_SRC

You will **not** see three MAP_WRITE slots as “mapped”; they are
host-visible buffers. The copy is the GPU event.

Firefox-style traces should be replayed **without** winit so RenderDoc can
attach to the player’s offscreen texture. This demo is native; attach
RenderDoc to the **demo**, not the player.

**Fidelity gaps.**

- Trace **does** record `Queue::write_buffer` bytes. RenderDoc shows the
  dest buffer **after** the pending-write flush at submit. Same data,
  different time.
- Trace **does not** record which ring index was picked. RenderDoc shows
  which buffer handle was `COPY_SRC` that submit.
- Player FPS is meaningless. RenderDoc GPU duration is real but includes
  validation layers if enabled.
- Trace + player can reproduce a validation panic on a laptop without a
  4090. RenderDoc of a 4090 frame will not replay on Intel the same way
  (tiling, present mode).

**Policy.** Keep both off the default `make ring` path. Do not bump wgpu
just to get traces.

- Validation / alignment / “share this encode” → 8-frame **trace**,
  feature-gated (`QGA_WGPU_TRACE=...`, `Trace::Off` otherwise). Not on
  wgpu 24.
- Picture / pipeline / barrier / “why empty” → **RenderDoc** on windowed
  FIFO/mailbox.
- Counters (`ring_copies`, `particle_fallbacks`, `static_uploads`) stay
  in-process. Neither tool replaces them.

Do not zip a 300-frame dirty trace (128 KiB × 300 blobs). Cut to 8. Zip
the folder, not a `.rdc`. Do not expect the player to explain a mailbox
tear. Do not expect RenderDoc to print `COPY_BYTES_PER_ROW_ALIGNMENT`.

### wgpu-hal Vulkan on this box (Software fact)

On this box wgpu is `ash` talking to the 4090. `wgpu-hal::vulkan` is the
unsafe portable layer: no validation, no usage tracking, persistent maps,
explicit barriers. `wgpu-core` is what turns your one encoder + one submit
into those HAL calls.

`wgpu-hal/src/vulkan/` (wgpu-hal 24.0.4):

| File | Role |
|------|------|
| `instance.rs` | `VkInstance`, layers, surface extensions; swapchain types live here + `mod.rs` (no separate `swapchain/` dir) |
| `adapter.rs` | `VkPhysicalDevice`, features, **`family_index = 0` assumed graphics** (`//TODO`) |
| `device.rs` | `VkDevice`, `gpu-alloc`, create buffer/image, `map_buffer` |
| `command.rs` | `VkCommandPool` + active `VkCommandBuffer`, copies, barriers, draws |
| `conv.rs` | WebGPU usages ↔ `VkAccess` / `VkPipelineStage` / `VkImageLayout`; present-mode map |
| `mod.rs` | `Api` static dispatch, `Queue`, `Buffer` / `Texture` |

Traits are static-dispatch (`vulkan::Api`). wgpu-core holds `Arc` so Vulkan
destroy-order holds. `Queue::submit` is not a second `VkQueue`; it waits
on surface-semaphore locks and a relay-semaphore mutex. You still have
**one graphics queue** (`family_index = 0`). “Overlap” is CPU record vs
last GPU frame, not a transfer queue.

**Buffers and mapping (the ring).** HAL maps **persistently**.
`Device::map_buffer` → `gpu-alloc` `block.map` → `vkMapMemory` returns
`BufferMapping { ptr, is_coherent }`. The pointer can stay valid while the
GPU reads if you barrier correctly. `unmap_buffer` is `vkUnmapMemory`.
Non-coherent memory needs `flush_mapped_ranges` /
`invalidate_mapped_ranges`. NVIDIA host-visible staging is usually
coherent; wgpu-core still calls flush on unmap when the flag says so.

WebGPU `map_async` is **not** HAL. Core waits on the last submit that used
the buffer, then calls HAL map (or already had it mapped at creation).
The 3 slots are three `VkBuffer`s with `HOST_VISIBLE` + `TRANSFER_SRC`.
Dest VB is `DEVICE_LOCAL` + `VERTEX` + `TRANSFER_DST`.
`MAPPABLE_PRIMARY_BUFFERS` would be a different memory type; this crate
does not request it. `gpu-alloc` failures become `DeviceError::OutOfMemory`
/ `Lost`, not `Validation`.

**Commands and barriers.** One HAL encoder ≈ one `VkCommandPool` with
recycled command buffers (`free` / `discarded`). `begin_encoding` gets a
CB; `end_encoding` ends it.

Copies: `copy_buffer_to_buffer` → `vkCmdCopyBuffer`;
`copy_texture_to_buffer` → `vkCmdCopyImageToBuffer` with
`VkBufferImageCopy`. `buffer_row_length` is `block_width * (bytes_per_row
/ block_size)` in **texels**. The **256 B pitch was already enforced in
core**. HAL trusts the numbers.

Barriers are explicit and cheap to miss if you used HAL raw:

```text
transition_buffers  → vkCmdPipelineBarrier + VkBufferMemoryBarrier (WHOLE_SIZE)
                      (src seeded TOP_OF_PIPE, dst BOTTOM_OF_PIPE so the mask is never null)
transition_textures → VkImageMemoryBarrier + layout from/to
```

Core’s usage tracker emits `BufferBarrier { from: COPY_SRC, to: MAP_WRITE }`
etc. This crate’s frame is:

```text
[pending write_buffer flush at submit start]
barrier: VB COPY_DST
vkCmdCopyBuffer  ring slot → VB
barrier: VB VERTEX
render pass   color COLOR_ATTACHMENT_OPTIMAL
end pass
barrier: color → TRANSFER_SRC_OPTIMAL     // if capture
vkCmdCopyImageToBuffer
submit
present barrier → PRESENT_SRC_KHR         // windowed
```

**Textures vs Vulkan images.** HAL textures are optimal `VkImage`s. No
linear tiling API. Layouts live only in `conv` + barriers (see the tiling
section above). Capture never maps the image; it maps the **buffer** after
the copy. Swapchain images are a separate object; headless has
`surface=false` and never touches `PRESENT_SRC`.

**Queue and present.** `Adapter::open` takes queue family 0 if it has
`GRAPHICS`. One `VkQueue`. `submit` takes command buffers + a fence.
Present uses mailbox/FIFO from surface caps; unrecognized mode
`1000361000` is `log::warn!("Unrecognized present mode {:?}", mode)` in
`conv.rs` `map_vk_present_mode` and ignored — that log line. No
compute-only queue, no transfer-only queue. Particle copies and draws
share that queue. A second `submit` would still be the same `VkQueue`.

**What this crate should not do.**

- Call `wgpu-hal` directly. You lose tracking; you must emit every barrier.
- Expect two Vulkan queues to hide `map_async` latency.
- Treat HAL persistent map as “skip `map_async`.” Core’s map state machine
  is what makes `ready[]` legal.
- Use linear images for capture. HAL will not give you
  `vkGetImageSubresourceLayout` for the color target.

RenderDoc attaches at this layer: you see the `vkCmd*` that `command.rs`
recorded. The RON trace stops one level up. Validation errors stop
**above** HAL. If HAL returns `DeviceError`, that is OOM/lost/internal,
not `UnalignedBytesPerRow`.

For qga-gpu the Vulkan backend is already the right shape: one queue,
DEVICE_LOCAL VB, HOST_VISIBLE ring, implicit barriers from one encoder.
The knobs you own stay in core-facing API (slot count, pitch helper, no
`Wait` on write maps).

### gpu-alloc is not the particle ring (Software fact)

“gpu-alloc” in this stack is the Vulkan `VkDeviceMemory` suballocator
inside **wgpu-hal**, not the 3-slot ring. You never pick buddy vs linear.
You pick WebGPU usages; HAL maps that to a memory type and a suballoc
strategy.

Two crates show up in the lineage:

- **zakarumych/gpu-alloc** — wgpu-hal **24 Vulkan** (`device.rs`
  `GpuAllocator`, `UsageFlags::HOST_ACCESS` / `FAST_DEVICE_ACCESS`). This
  crate.
- **Traverse-Research/gpu-allocator** — wgpu-hal **DX12** today
  (`MemoryLocation::GpuOnly` / `CpuToGpu` / `GpuToCpu`). Planned for Vulkan
  ([gfx-rs/wgpu#5925](https://github.com/gfx-rs/wgpu/issues/5925)); later
  wgpu Vulkan `device.rs` already uses it.

Same idea: few `vkAllocateMemory` objects, many `VkBuffer`s bound at
offsets. `maxMemoryAllocationCount` on NVIDIA is thousands, not unlimited
(`DeviceProperties.max_memory_allocation_count` in the HAL config).

**This crate is `MemoryHints::Performance`.** wgpu-hal 24 `adapter.rs`
`perf_cfg` (not the old gfx-hal `alloc.rs` numbers):

| Knob | wgpu-hal 24 Performance | Old gfx-hal default | Meaning |
|------|-------------------------|---------------------|---------|
| `dedicated_threshold` | 32 MiB | 32 MiB | ≥ this → own `VkDeviceMemory` |
| `preferred_dedicated_threshold` | **1 MiB** | 8 MiB | prefer dedicated if the type allows |
| `transient_dedicated_threshold` | 128 MiB | 128 MiB | staging/transient dedicated only if huge |
| `starting_free_list_chunk` | 128 MiB | `linear_chunk` 128 MiB | bump/linear slab size |
| `final_free_list_chunk` | 512 MiB | — | grow the linear slab up to this |
| `minimal_buddy_size` | **1 B** | 1 KiB | smallest buddy leaf |
| `initial_buddy_dedicated_size` | 8 MiB | 8 MiB | first buddy block of device-local heap |

`MemoryHints::MemoryUsage` shrinks those (dedicated 8 MiB, linear start
8 MiB). Do not switch this crate to MemoryUsage to “save” 128 KiB slots.

**Dedicated.** One resource ↔ one `VkDeviceMemory`. Best for large images
(1920×1080 color, bloom mips) and anything the spec wants
`DEDICATED_MEMORY_REQUIRED` for. Worst for 32 B records.

**Buddy.** Power-of-two split/merge in a big block. Good for many medium
DEVICE_LOCAL buffers (VB, UBO, capture buffer). Internal fragmentation:
a 128 KiB slot may consume 128 KiB exactly; 129 KiB consumes 256 KiB.

**Linear / bump.** Fast allocate, no general free. Fits
`Queue::write_buffer` scratch and belt chunks that die at submit. A 128
MiB linear chunk is why a belt that `finish`es every frame should
**recall**, not leak chunks: the HAL slab is huge, but mapped windows and
`vkMapMemory` count still matter.

gpu-allocator’s public labels (DX12 now; Vulkan after #5925):

| `MemoryLocation` | Vulkan property intent | This crate |
|------------------|------------------------|------------|
| `GpuOnly` | `DEVICE_LOCAL`, not host-visible | dest VB, color/bloom images |
| `CpuToGpu` | `HOST_VISIBLE` (+ ideally `HOST_COHERENT`) | ring slots, `write_buffer` staging |
| `GpuToCpu` | host-visible cached | capture MAP_READ |

wgpu-hal 24 Vulkan does the same split with `gpu_alloc::UsageFlags`:
`MAP_WRITE`/`MAP_READ` → `HOST_ACCESS` + `UPLOAD`/`DOWNLOAD`; else
`FAST_DEVICE_ACCESS`. `linear: true` is a gpu-allocator buffer flag
(always for buffers). Images are non-linear and often dedicated.

**How `create_buffer` chooses (wgpu-hal 24 Vulkan).**

1. `vkCreateBuffer` with usage bits from `conv::map_buffer_usage`.
2. `vkGetBufferMemoryRequirements` → size, alignment, `memoryTypeBits`.
3. Filter types HAL allows (`valid_ash_memory_types`).
4. Pick `UsageFlags` from MAP_READ / MAP_WRITE / none.
5. `GpuAllocator::alloc`; `vkBindBufferMemory(memory, offset)`.
6. Map path: gpu-alloc tracks one map per `MemoryBlock`. Persistent map
   at HAL; core still `map_async` at WebGPU.

OOM is `gpu_alloc::AllocationError` → `DeviceError::OutOfMemory` /
`Lost` (`TooManyObjects` included). wgpu 24 Vulkan does **not** pre-check
a % of `VkMemoryHeap` budget; that `error_if_would_oom_on_resource_allocation`
path is later wgpu. NVIDIA still OOMs for real.

ReBAR / “mappable device-local”: if a type is both `DEVICE_LOCAL` and
`HOST_VISIBLE`, HAL *may* put `HOST_ACCESS` there. You still must not set
`VERTEX|MAP_WRITE` without `MAPPABLE_PRIMARY_BUFFERS`. Two allocations:
128 KiB host-visible + DEVICE_LOCAL VB is the portable 4090 path.

**vs this crate’s ring.**

| Allocator | Grain | Lifetime |
|-----------|-------|----------|
| gpu-alloc (Vulkan 24) / gpu-allocator (later) | `VkDeviceMemory` slabs (KiB–MiB) | process / device |
| Particle ring | 3 × 128 KiB WebGPU buffers | app-owned, reused |
| StagingBelt | bump inside those buffers | per submit |
| `Queue::write_buffer` | HAL transient linear + copy | until submit flush |

Growing a slot (`×2`) is a **new** `create_buffer` + new suballoc + bind.
`particle_grows` is the only counter you need. Do not grow every frame:
that is a new buddy split every time and leaves the old block to the
allocator’s free list (or dedicated destroy).

Capture buffer `padded_bpr × height` (~8 MiB at 1920) sits near
`preferred_dedicated_threshold` (1 MiB on this Performance config, so it
may be dedicated). One dedicated or one buddy slab is fine. Do not
suballocate capture rows yourself inside gpu-alloc — that is the 256 B
pitch buffer you already own.

**What not to tune from qga-gpu.**

- Do not depend on `gpu-alloc` / `gpu-allocator` in this crate.
- Do not request dedicated memory via HAL.
- Do not pack the UBO, three ring slots, and VB into one `VkDeviceMemory`
  by hand.
- Do not assume buddy leaves match WebGPU `MAP_ALIGNMENT` 8; core pads
  requirements first. wgpu-hal 24 `minimal_buddy_size` is 1 B, not 1 KiB.

If `create_buffer` starts returning OOM on grow, you leaked slots (old cap
not dropped) or filled the HOST_VISIBLE heap with 300 one-off belt chunks.
`particle_grows=0` on `make ring` means the HAL allocator is in steady
state: three mapped blocks + one VB + one color + one readback. That is
the strategy you want.

## Upload contract (from inner_cone)

1. Tessellate static topology once (parametric sphere / cone / torus).
2. `retain_static_fibers` / `retain_meshes` / `upload_hubs` when the lattice
   key changes. Identical re-upload is skipped (`static_skipped`).
3. Each frame: `write_live_fibers` (live harmonics), `write_particles`,
   `write_hud`, `render`.
4. Tubes: `GpuFiberPoint` centerlines, camera-facing extrusion in `fiber.wgsl`.
   No CPU ribbon rebuild on camera/time.
5. Particles: 3-slot `MAP_WRITE` ring. Prefer any ready slot. Fallback
   `write_buffer` only if zero slots ready. Do not drop `pending`.
6. Rebuild only aperture/height/zener-driven topology (caller sets
   `mark_static_dirty`). Camera and particles do not realloc fibers.

## Key decisions

1. **No default `qga-math` / `qga-sim` dependency.** GPU records only. Callers
   convert. Optional `--features qga-math` for `From<&qga_math::Fiber>`.
2. **No scene-kind draw gate.** `render` draws whatever was uploaded.
   inner_cone today passes `Reveal` so hubs/HUD draw; that leak is not reproduced.
3. **Frame uniforms are 256 bytes.** Cosmos palettes rewrite `GpuParticle` hue
   in `qga-app::convert`; they are not a 272-byte UBO pad.
4. **Shader tube extrusion** instead of CPU `RibbonVert` (48 B).
5. **Particle staging ring** instead of realloc + `write_buffer` on every path.
   Not `StagingBelt`: one 128 KiB blit/frame is the wrong shape for a belt.
6. **Headless actually renders** to an offscreen color target. The engine
   returned `Ok(None)` with no surface.
7. **Parametric mesh helpers** are tessellation (Software fact), not observer
   / Zener / flux meaning.
8. **HUD primitives only.** Scene overlays stay in the caller.

## Features

`default = ["winit"]`. Optional: `headless`, `capture`, `glow`, `qga-math`.

## Non-goals (v0)

- Editing or opening a PR into `qga_engine` or `inner_cone` in this extract.
- Porting `qga-app` scenes (lab / realm / cosmos / oam / reveal).
- Porting `qga-sim` worldgen, n-body ICs, or OAM twist PDE.
- CUDA, OpenGL, WebGPU-in-browser.
- N-body compute, meshlets, DLSS/FSR.
- Persistent `vkMapMemory` / BAR-mapped particle VB through wgpu.
- `StagingBelt` on the particle path.
- Claiming the Z-map or 350/π as theorems inside the renderer.
- Vendoring all of `qga-math`.

Consumer wiring: [MIGRATION.md](MIGRATION.md). `inner_cone` @ `89a890c`
git-pins `qga_engine@7e7866b` + `qga_gpu@b9c9994` (`features = ["capture"]`).
[`qga_engine`](https://github.com/qga-lab/qga_engine) @ `7e7866b` git-depends
with `rev = "b9c9994"` (`Cargo.lock` is not the pin). No `v0.1.0`. Do not
enable `qga-math` on this crate for those callers.

## Claim labels

Theorem / Model / Software fact / Open. Everything in this crate is
**Software fact** unless a comment says otherwise.
