---
title: "Home"
weight: 10
---

# go-quake1 documentation {#go-quake1-documentation}

**A pure-Go [Quake](https://en.wikipedia.org/wiki/Quake_(video_game)) 1 (id Tech 1,
1996) engine** for bare-metal TamaGo + UEFI under the
[cloud-boot](https://github.com/cloud-boot) loader — **hand-ported** from the
[tyrquake](https://github.com/sezero/tyrquake) NetQuake (single-player) branch
(commit `6531579`) and wrapped in cloud-boot virtio adapters.

It is the sibling of [go-doom/engine](https://github.com/go-doom/engine) in the
DOOM → Quake roadmap. The family also reserves
[go-quake2](https://github.com/go-quake2) (id Tech 2) and
[go-quake3](https://github.com/go-quake3) (id Tech 3).

## Status {#status}

**Phase Q-1a in flight** — hand-porting tyrquake-NQ to pure Go one module at a
time, with parity tests against the upstream C behaviour. Every ported package
holds **100% test coverage** (the org gate) and validates on all six 64-bit
architectures.

| Component | State |
| --- | --- |
| Porting conventions + `reference/` tyrquake mirror (`6531579`) | done |
| **common** (`mathlib`, `crc`, `cmd`, `cvar`, `zone`, `qstr`, `sizebuf`, `qpath`, `qparse`, `msg`, `qargs`, `pak`, `vfs`, `wad`, `keys`, `protocol`, `anorms`) | done — 17 packages, 100% cov |
| **models** (`bspfile`, `mdl`, `spr`, `model` dispatcher) | done — 4 packages, 100% cov |
| **bsptrace** (`Mod_HullPointContents` + `Mod_TraceHull`) | done |
| **progs** (QuakeC VM — 65 opcodes + edicts + parser + math builtins) | done — 100% cov |
| **server** (host + `sv_world` + `sv_main` + `sv_phys` + `sv_user`) | in progress |
| **client** + soft renderer (`r_*`, `d_*`, `cl_*`) | pending |
| **sound** (`snd_dma` + `snd_mem` + `snd_mix`) | pending |

## Sound in the wasmbox desktop {#wasmbox-sound}

Quake runs as a window of the [wasmbox](https://github.com/wasmdesk/wasmbox) web desktop
in a Web Worker, which has no `AudioContext`. The wasmbox backend therefore writes the
mixer output (11025 Hz stereo) as float32 PCM into a lock-free `SharedArrayBuffer` ring
([`wasmbox/audioring`](https://pkg.go.dev/github.com/wasmdesk/wasmbox/audioring)), and
the wasmbox page plays it from an AudioWorklet, resuming on the first click or key. The
ring is 1024 frames (93 ms); what does not fit is dropped rather than blocking the game
loop.

## WebGPU presenter in the browser build {#webgpu-present}

The standalone browser build (`cmd/quake-wasm`) shows its frame through
[WebGPU](https://github.com/go-gfx/webgpu) when the browser has a hardware adapter. The
software rasterizer still draws the 8-bit palettized frame; the index buffer is uploaded as
an `r8uint` texture and the palette as a 256x1 texture, and a fragment shader does the
palette lookup and the scaling to the canvas, with one batched submit per frame. The CPU no
longer converts the frame to RGBA or calls `putImageData`. Without an adapter (or with only
a software one such as SwiftShader) the page keeps the Canvas2D path.

A browser test reads the canvas back and compares every pixel with the CPU conversion; a
wrong palette index must make it fail. `quake-bench present` measures the Canvas2D path and
the WebGPU path in interleaved rounds on the same machine, discarding any round above a
1-minute load of 8. The figures, and what is still on the CPU (the BSP walk and the span
filling), are in the engine repository under `docs/webgpu-present/`.

## The BSP world on the GPU {#gpu-world}

The browser build can draw the BSP world itself on the GPU (`render/gpuworld`, on
[go-gfx/webgpu](https://github.com/go-gfx/webgpu)). The CPU still walks the BSP and applies the
PVS (`bsprender`); the GPU then draws the selected faces with their base textures and lightmaps
into an 8-bit `r8uint` frame of palette indices, through the colormap, exactly as the software
filler does per pixel. The presenter above can display that frame unchanged, so the look stays
the software one.

It is checked against a software reference of the same content (`worldview`) on 16 fixed
`e1m1` views on a real GPU (Apple, not a software adapter): 94.9% of the 1,024,000 pixels are
identical and 1.52% are neither identical nor a one-pixel shift of the reference (tolerance
2.5%; a damaged colormap fails the check). Removing the textures leaves 0 unexplained pixels, so
the remaining differences are texel selection (the software filler is only approximately
perspective-correct). The check needs the shareware `pak0.pak`, which is never in the
repository. The reference draws in the engine's walk order with no depth test and no back-face
cull, so where faces overlap the picture depends on the draw order (reversing the order changes
19% of the pixels): the GPU path matches it on purpose, and hidden-surface handling like GLQuake's
is a later step. Dynamic lights and animated lightstyles are also later steps. Figures:
`docs/webgpu-world/` in the engine repository.

### Alias models, sky and water on the GPU

Since v0.4.0 the GPU also draws, after the world and in the same 8-bit frame, the alias models
(monsters, items and a view weapon), the sky and the water (`*` textures). Each is compared with
the engine's own software fill on 16 fixed `e1m1` views, over the region the part covers, with a
tolerance stated before the first measurement (3% of pixels not explained by a one-pixel
shift) and a planted corrupted texture that must fail the check:

| part | unexplained | software alone, 1280x800 | GPU wall time alone |
|---|---|---|---|
| alias models | 0.10% | 0.595 ms | 0.635 ms (not faster) |
| sky | 2.32% | 6.12 ms | 0.585 ms |
| water | 0.05% | 8.42 ms | 2.0 ms |

The GPU wall time includes a floor of about half a millisecond, so the alias figure is not
resolved from the CPU one; GPU time itself is not measured yet. Figures and methods:
`docs/webgpu-world/2026-11-03-*.md` in the engine repository.

### The software reference is now Quake's own (v0.5.0)

Those first references were the engine's own approximations, not tyrquake's: its sky was not the
dome, its water not the `Turbulent8` of `d_scan.c`, its alias light a per-triangle mean. The
engine's software renderer now ports the originals, with tests pinned on values from the source
(the `sintable` entries, the sky coordinates of known pixels, the lighting setup of
`R_AliasSetupLighting` and every entry of `anorms.h`): water is `Turbulent8` with
`D_CalcGradients` and 16-pixel sub-spans, the sky is the `R_MakeSky` dome of `d_sky.c`, and
aliases are Gouraud-lit from the light normals. The GPU is compared with those, on the same 16
views and the same tolerances:

| part | unexplained against Quake's own | before |
|---|---|---|
| water | 0.13% | 0.05% (against the old approximation) |
| sky | 0.30% | 2.32% |
| alias models | 0.10% | 0.10% |
| the BSP world | 0.006% | 1.52% |

The world figure comes from isolating the old mismatch: the reference's 8-pixel perspective
sub-spans and its left-edge pixel coverage, which the shader now reproduces (the GPU world costs
about 0.15 ms more at 1280x800). Not ported in v0.5.x: the light at an entity's origin
(`R_LightPoint`; entities used a constant), the fixed-point edge walk of `d_polyse.c`, the surface
cache and its mip levels; and the GPU did not reproduce the 16- and 32-pixel sub-spans of water and
sky.

### Entity light, the alias edge walk and sub-spans (v0.6.0)

Three of those four are now done, each with tests pinned on values worked from the source, the
GPU re-compared on the same 16 views and the same tolerance, and a planted defect that must fail:

| piece | unexplained against Quake's own | before |
|---|---|---|
| `R_LightPoint` (entity light from the lightmap; hand-worked bilinear cases) | 0.10% (alias) | the same figure at a constant light of 76; lighting the GPU at 76 against `R_LightPoint` is 56.0% |
| `d_polyse.c` fixed-point edge walk (all 1,024 `adivtab.h` entries, a hand-worked triangle, 3,000 random triangles against a derivation) | 0.75% (0.03% without far models) | 0.10% against the older float scanline; the shader taking the floor for the ceiling is 7.7% |
| 16-pixel water and 32-pixel sky sub-spans on the GPU | sky 0.02% (differing 0.08%), water 0.11% (differing 0.62%) | sky 0.30% (4.90%), water 0.13% (2.50%) |

The edge walk also fixed a defect: the old alias paths drew the triangles Quake culls, so models
were drawn inside out. The GPU does not reproduce the recursive drawer Quake uses for a distant
model (that is the 0.75% against 0.03%). Still not ported: the clipping of triangles that cross the
near plane (`r_aclip.c`).

### The surface cache with mip levels (v0.7.0)

The fourth piece. The world is no longer sampled per pixel from the full-size texture and the
lightmap: each face is drawn from a lit surface at the mip level its distance selects, as in
Quake's `D_CacheSurface`, `R_BuildLightMap`, `R_DrawSurface` (with its four block drawers) and
`D_MipLevelForScale`, the allocator `D_SCAlloc` included, with tests pinned on values worked by
hand from the source. The GPU bakes the same surfaces at four levels into one atlas and reads the
level the CPU chooses per face and frame.

| comparison, 16 views | differing | unexplained |
|---|---|---|
| the v0.6.1 reference against the surface-cache reference | 40.47% | 23.86% |
| the GPU against the surface-cache reference | 0.006% | 0.0006% |
| the GPU without the mip choice (level 0) | 27.57% | 13.68% (fails the 2.5% tolerance) |
| the GPU one mip level too far | 57.93% | 28.20% (fails) |

`nearzi`, the nearest depth of a surface, exists only in Quake's edge-based hidden-surface pass;
the stand-in is the face's polygon clipped to the view frustum. The engine's own world loop still
bakes one full-size surface per face (engine issue 72). v0.7.1 cut the GPU path's CPU cost for
the mip choice from 0.280 ms to 0.205 ms (v0.6.1: 0.180 ms) with one transform per face, a
bounding-sphere shortcut and the level carried on the draw's first instance, at the same 0.0006%.
Details, timings against v0.6.1 and what remains:
`docs/webgpu-world/2026-11-04-surfcache.md` in the engine repository.

## Run it in the wasmcontainers playground {#playground}

From engine v0.10.0, `cmd/quake-wctr` is the engine as a wasip1 guest. It draws every frame
with the software renderer and presents it on a `wasi-gfx:surface`: the 8-bit frame and its
palette go to the GPU as storage buffers, and a loop-free WGSL shader looks up the colours
(`render/wasipresent`). Keys come from the surface (`backend/wasisurface`). There is no audio
yet, and `SkipMenu` boots straight into `e1m1`.

The [go-wasmcontainers playground](https://go-wasmcontainers.github.io/playground/) has a
`quake` image that runs it in a container on a GPU:

```sh
wctr run --rm --gpu -m 512m quake
```

That plays `e1m1` of LibreQuake. Its data is [go-quake1/freepak](https://github.com/go-quake1/freepak)
v0.2.0, a sovereign `pak0` with no id Software bytes. The image is GPL-3.0-or-later and names its
source in `/usr/share/doc/quake/copyright`.

The playground's end-to-end test, `TestQuake`, runs in CI on a real GPU (GitHub's macOS runner,
Apple adapter). For fixed view 4 (640x400), the frame the page receives has RGBA sha256
`f84421f86941b51dbb5afafd0eb2cb6a608314e3064b07100ff83c2fb2b3c8df`, the exact figure the guest
computed on the CPU. 150 frames reach the page, 140 of them distinct. One CI run measured 58 ms
per frame on that runner; that is one run, not a benchmark.

### Level 2: the world drawn on the GPU in the container {#level-2}

From engine v0.11.0 to v0.13.0 the GPU world renderer (`render/gpuworld`) also runs inside the
container, on the batched stream (`wasi_webgpu_batch`): one host crossing per frame instead of one
per draw.

- **v0.11.0, build for wasip1.** `render/gpuworld` builds for `GOOS=wasip1` on a remote device of
  go-gfx/webgpu. The atlases go up with `Recorder.WriteTexture` in bands of at most 8 MiB of whole
  rows, because the 33 MB surface atlas is above the host's 32 MiB frame and `Queue.WriteTexture`
  is refused on a remote device. A frame is read back with `Recorder.CopyTextureToBuffer` (rows padded
  to 256 bytes, removed on the CPU).
- **v0.12.0, bounded loops.** The batch host runs a render module only if every loop is a
  constant-bound leaf loop of at most 64 iterations. The world shader's polygon-edge loop and the
  sky and water `runOf` loop now run to `MaxPolyVerts` (64) and break at the face's vertex count;
  `Build` refuses a face with more vertices. The frame is unchanged: in Chromium on the Apple GPU the
  world check differs in 11 of 256,000 pixels, as before.
- **v0.13.0, `quake-wctr --worldref N --gpuworld`.** The view is drawn on the batch device, the
  index frame is read back in the same submit and presented through `wasipresent`. A GPU frame is not
  hash-equal to the CPU's, so it is compared with the software frame at the world check's measure,
  the pixels a one-pixel shift does not explain (tolerance 2.5 %), not with a SHA-256. In the
  playground's `TestQuake` on the Apple GPU: 7 of 256,000 pixels differ, 0 unexplained. The
  `--worldref` hashes `TestQuake` compares are still the software frame's, unchanged.

This needs go-wasmcontainers/gpu v0.13.0 (go-gfx/webgpu v0.18.0, with its draw quota) and the
playground's engine v0.13.0 or later.

- **v0.14.0, timing.** `quake-wctr --worldref N [--gpuworld] --frames K` prints the first frame apart
  (the GPU's device, map build and atlas upload) and the mean, median, min and max wall-clock time of
  the other K frames, present excluded. Measured in Chromium on the Apple GPU (freepak e1m1, view 4,
  8 interleaved rounds, load at most 8): 7.68 ms per frame in software against 1.08 ms on the GPU
  with its read-back (-85.9 %), after a 164 ms first frame.
- **v0.15.0, the game loop.** `quake-wctr --gpuworld` (no `--worldref`) plays with the lit world
  faces drawn on the GPU, one submit with the read-back per frame; the CPU stays the default. The
  software pass draws over it, in its own order, the sky and water (the GPU sky is not Quake's dome),
  the entities, the particles, the view model, the HUD and the menu. The device is opened and the atlases
  uploaded once, before the first frame. A map whose surfaces do not fit the 8192 x 8192 atlas (freepak
  e1m2, e1m3, e1m6) plays on the CPU. The host's draw quota (1 Mi per draw, 16 Mi per stream) holds with
  room: at most 38,571 indices a frame and 27 per draw on the freepak e1 maps. With `--gpuworld-check`
  every frame is also drawn on the CPU and compared at the world check's measure: 30 frames in the
  playground's `TestQuake`, worst frame 0.49 % unexplained against 2.5 %.

Light styles and the animation of `+` textures stay at their baked values on the GPU faces.

## Licensing {#licensing}

The repository is a **BSD-3-Clause wrapper** around the **GPL-2.0 engine** code
(id Tech 1 is GPL): the cloud-boot adapters and tooling are BSD-3, while the
ported engine retains the upstream GPL-2.0 license — see the
[engine repository](https://github.com/go-quake1/engine).

## Where to go next {#where-to-go-next}

- Source: [github.com/go-quake1/engine](https://github.com/go-quake1/engine)
- The shared boot architecture and the DOOM → Quake roadmap live in
  [cloud-boot/docs](https://cloud-boot.github.io/docs/).
