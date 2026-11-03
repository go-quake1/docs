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
about 0.15 ms more at 1280x800). Not ported: the light at an entity's origin (`R_LightPoint`;
entities use a constant), the fixed-point edge walk of `d_polyse.c`, the surface cache and its
mip levels; and the GPU does not reproduce the 16- and 32-pixel sub-spans of water and sky.

## Licensing {#licensing}

The repository is a **BSD-3-Clause wrapper** around the **GPL-2.0 engine** code
(id Tech 1 is GPL): the cloud-boot adapters and tooling are BSD-3, while the
ported engine retains the upstream GPL-2.0 license — see the
[engine repository](https://github.com/go-quake1/engine).

## Where to go next {#where-to-go-next}

- Source: [github.com/go-quake1/engine](https://github.com/go-quake1/engine)
- The shared boot architecture and the DOOM → Quake roadmap live in
  [cloud-boot/docs](https://cloud-boot.github.io/docs/).
