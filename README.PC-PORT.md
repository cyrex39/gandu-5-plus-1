# gandu 5+1 on PC (NVIDIA) — VIctory fork with our fixes

This is a fork of [fxceless/victory-fork](https://github.com/fxceless/victory-fork)
(which builds on [Marvisak/VIctory](https://github.com/Marvisak/VIctory)) carrying
extra fixes for running the Xbox Series build of gandu 5+1 on an **NVIDIA RTX 4080 +
AMD Ryzen 9 7900X3D** PC. **No game files, keys or assets are here.** You need your
own copy of the game. Licensed GPL-3.0-or-later like upstream; see LICENSE and the
credits at the bottom of the upstream README.

## Known issues

- **The mosaic texture issue is NVIDIA-specific.** On AMD Radeon GPUs the
  Radeon-tuned lines (the `victory-world`/DexNom builds) show the world with
  complete textures; on NVIDIA the same builds show blocky, mosaic-like light
  and texture patches. Two causes are identified and fixed in this fork:
  an indirect compute dispatch cut off at 65,535 groups (fixed by
  **dispatches in parts**, below) and 84 texture descriptions laid out
  differently from Microsoft's own library (fixed by the **exact XG texture
  layout**, below). On AMD neither shows up the same way, because the AMD
  lines carried these fixes first and AMD's driver is less strict about
  wave64 compute pipelines. Remaining NVIDIA-side gaps: ~15 wave64 compute
  pipelines the NVIDIA driver refuses (about 300 draws a frame) and 5
  ray-structure-builder programs that need 64-lane waves, which NVIDIA GPUs
  do not execute.
- **Shadowed and indoor areas are black.** The title's ray-traced lighting
  (bounce light, sun and lamp shadows) does not run yet on any GPU.
- **Performance is far from the console.** See the table above; the
  bottleneck is the translation layer's CPU-side memory copying, not the GPU.

## Measured results (2026-11-25, this machine)

| Stage | World frame time | FPS |
|---|---|---|
| Upstream fork tip, defaults | 190–230 ms | 4.4–5.2 |
| + speed switches (see launcher) | 140–173 ms | 5.8–7.1 (up to 14.7 in light rooms) |
| + everything below | **102–147 ms** | **6.8–9.8** |

Menus and cutscenes run at the game's native 30 fps. Full shader archive pre-warmed
(25,696 of 25,993 translated; 297 refused) — `victory translate <game dir> --warm all --jobs 20`.

## What we changed on top of upstream

- **Dispatches in parts** (`bound_split`): an indirect compute dispatch asking for
  more than 65,535 groups in x (the title's final gather of indirect light asks
  ~115,200 — 8 groups per 16×16 tile of 2560×1440) used to be cut off; the tiles
  past the cut kept stale light (mosaic pattern). Now written as two dispatch
  records, with the shader told its true first-group number via a new
  `USER_GROUP_BASE` root constant. Ported from the DexNom/Radeon line
  (community fork of the `victory-world` line), hardware-neutral.
- **XG texture layout answered exactly like Microsoft's `xg_xs.dll`** (from
  AnyXboxX lot 9): tail mips in parent mode with a position, 64 KiB tilings
  indexed by subresource. Upstream's answers differed from the real library for
  84 texture descriptions.
- Cherry-picked from the fork's assembled lines (LooperSalty/victory, AnyXboxX):
  BCPack BC7 first-entry indices (#374), MIMG 3D Z-offsets / a16 / cubemap LOD
  (#353, #386), dual sampler slots with/without comparison, `v_div_fixup_f32`
  Infinity fix (#349), exact page give-back under placements (#358),
  refuse in-place placement under late waits (#358 follow-ups) — which makes
  `VICTORY_LATE_WAIT` safe to use again, and the upstream gather
  (`VICTORY_INDIRECT_GATHER`, 10.3 fps in the first rooms upstream).
- Switches known to cause visual bugs are used knowingly: `VICTORY_RELAY=2`
  (fork issue #89: skin) and `VICTORY_MERGE_GAP` (not exact) are the two to
  drop first if anything looks wrong.

## Build and play

```powershell
cargo build --release -p victory
VICTORY_GAME_DIR="<game folder>" VICTORY_CACHE_DIR="<cache>" VICTORY_DATA_DIR="<saves>" \
  target\release\victory.exe run "<game folder>\game_scarlett_master.exe" --execute
```

Requirements (upstream): Windows 10/11 x64, AVX2 CPU, D3D12 GPU with SM 6.6 and
mesh shaders (RTX 20 series or newer), Rust 1.98.1 (pinned), several GB of
Windows commit (raise the paging file if `crashes.log` reports error 1455 —
this build needs ~33 GB commit in the world).

First run translates shaders (slow); warm the cache first with the `translate`
command above. The title's ray-traced lighting does not run yet on any GPU
(interiors/shadows black); on NVIDIA the RT structure builder is additionally
blocked by 5 builder programs that need 64-lane waves, which NVIDIA GPUs do not
execute. Credits: Marvisak (VIctory), fxceless and fork contributors,
ZSlayerHQ/victory-world line, AnyXboxX (LooperSalty), DexNom, Yorivel, AscilCH,
JordanFableFur, HailLucifer666, davutsimsekk, cxchhh, myokex (victory-amd),
ivall (XCL) — see THIRD_PARTY.md in the AnyXboxX repository for the full list.
