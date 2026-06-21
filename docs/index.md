# go-quake1 documentation

**A pure-Go [Quake](https://en.wikipedia.org/wiki/Quake_(video_game)) 1 (id Tech 1,
1996) engine** for bare-metal TamaGo + UEFI under the
[cloud-boot](https://github.com/cloud-boot) loader — **hand-ported** from the
[tyrquake](https://github.com/sezero/tyrquake) NetQuake (single-player) branch
(commit `6531579`) and wrapped in cloud-boot virtio adapters.

It is the sibling of [go-doom/engine](https://github.com/go-doom/engine) in the
DOOM → Quake roadmap. The family also reserves
[go-quake2](https://github.com/go-quake2) (id Tech 2) and
[go-quake3](https://github.com/go-quake3) (id Tech 3).

## Status

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

## Licensing

The repository is a **BSD-3-Clause wrapper** around the **GPL-2.0 engine** code
(id Tech 1 is GPL): the cloud-boot adapters and tooling are BSD-3, while the
ported engine retains the upstream GPL-2.0 license — see the
[engine repository](https://github.com/go-quake1/engine).

## Where to go next

- Source: [github.com/go-quake1/engine](https://github.com/go-quake1/engine)
- The shared boot architecture and the DOOM → Quake roadmap live in
  [cloud-boot/docs](https://cloud-boot.github.io/docs/).
