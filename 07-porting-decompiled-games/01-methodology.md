# Methodology: the libultraship/Shipwright Pattern

Grounded in three real, shipped Vita ports — Rinnegatamante's **Ghostship** (Zelda 64: Ocarina of
Time / Majora's Mask, forked from HarbourMasters/Ghostship), **SpaghettiKart** (Mario Kart 64), and
**2ship2harkinian** (Zelda 64: Majora's Mask again, a separate lineage) — verified by reading their
actual `Makefile.vita` files and repository trees.

## The three-layer architecture

Every game in this family (the "Shipwright" family, named after the original Ship of Harkinian
project that started it) is built as three layers with a hard boundary between them:

1. **The decompiled game.** Matched C reconstructed from the original console binary via a
   community decompilation project. This layer calls what look like the original console SDK's
   functions (`osContInit`, `osViSetMode`, and the like for N64 titles) but never touches a host
   platform API directly.
2. **libultraship.** A shared, actively-maintained portability library that implements those SDK
   entry points against real host APIs: SDL2 for windowing/input/audio, and **Fast3D** for
   translating the console's display-list graphics commands (F3DEX2 for N64) to a real GPU API —
   OpenGL, D3D11/12, Metal, or, on Vita, vitaGL (see [section 03](../03-vitagl/README.md)).
3. **An asset pipeline** (Torch, for this family) that extracts the original binary's data into
   platform-agnostic archives ahead of time, off-device, driven by YAML configs checked into the
   game's own repo. This layer is untouched by a Vita port — it runs once, on the developer's
   machine, before any device-side build.

The practical consequence: **porting to a new platform is a libultraship problem, not a
whole-game problem.** Give libultraship a Vita-flavored rendering backend, input backend, and OS
shim, and the decompiled game code above it — often several hundred thousand lines — needs zero
changes.

## The build-system pattern

All three reference ports share the same recipe, differing only in per-game specifics:

### A standalone `Makefile.vita`, not an integrated CMake toolchain

Every one of these games already ships a `CMakeLists.txt` for desktop platforms. None of the three
Vita ports extend that CMake build with a VitaSDK toolchain file — all three instead add a
**separate, hand-written `Makefile.vita`** next to the existing CMake build, invoked directly
(`make -f Makefile.vita`) rather than through `cmake --build`. This mirrors the reasoning already
documented for other large VitaSDK projects in this wiki (see
[02-vitasdk: project setup](../02-vitasdk/02-project-setup-build-system.md)): CMake's official
VitaSDK toolchain support gets fragile on codebases this large, and a plain Makefile sidesteps it
entirely.

The Makefile.vita itself follows a consistent shape across all three: a `TARGET` name and 9-character
`TITLE` ID, a source-file list assembled from `libultraship/src/...` (unchanged across every port —
this is the shared portability layer), the game's own Torch factory sources
(`Torch/src/factories/<game>/...`), and the game's own source tree. Link flags pull in the same
core Vita library stack every one of these ports needs:

- `-lvitaGL` — the GL-shaped rendering target Fast3D draws into (see [section 03](../03-vitagl/README.md))
- `-lvitashark -lSceShaccCgExt` — runtime shader compilation, since the Vita has no native GLSL
  compiler (see [03-vitagl/05-shaders.md](../03-vitagl/05-shaders.md))
- `-lSDL2` — reused as-is; libultraship already routes input/audio/windowing through SDL2 on every
  platform, and VitaSDK ships an SDL2 port compatible enough that this layer needs no game-specific
  changes
- `-lmathneon` — NEON-optimized math, standard across VitaSDK homebrew for anything doing real-time
  3D
- the usual `Sce*_stub`/`taihen_stub` set any VitaSDK homebrew links against

### A small OS/input shim replacing the handful of SDK entry points SDL2 doesn't cover

Ghostship's `lib/src/osViTable.c` is the clearest example: a few hundred lines reimplementing the
N64 `libultra` video-interface and controller-init functions (the ones the decomp layer still calls
directly) using the same SDL2 calls the desktop build already relies on, just built against
VitaSDK's SDL2 instead of desktop SDL2. This file is small and mechanical to write *once you know
which entry points need it* — the fastest way to find that list for a new game is to read its
desktop-platform equivalent of this file (often just called `os.cpp` at the repo root) and see which
functions it implements that libultraship doesn't already provide generically.

### LiveArea assets and packaging — nothing game-specific

Every reference port ships the same `livearea/` folder (`icon0.png`, a background image, a
`startup.png`, `template.xml`) and runs the same `vita-elf-create` → `vita-make-fself` →
`vita-pack-vpk` pipeline documented generically in
[02-vitasdk: homebrew app anatomy](../02-vitasdk/03-homebrew-app-anatomy.md). There is nothing about
this stage specific to being a decompiled game rather than any other homebrew.

## Best practices

- **Read the game's own desktop OS shim before writing the Vita one.** It's the fastest way to find
  the exact, minimal set of functions that need a Vita-side implementation, rather than guessing
  from the libultraship source at large.
- **Don't fight CMake for this family of project.** The established convention across every shipped
  port in this family is a standalone `Makefile.vita`, not a VitaSDK CMake toolchain file — that's
  a deliberate, repeated choice across independent ports, not an oversight worth "fixing" by
  integrating CMake properly.
- **Treat the asset pipeline as already solved.** If the game already has a working Torch/YAML
  asset-extraction setup for desktop, a Vita port touches none of it — don't spend planning time
  here.
- **Expect the link-and-fix loop to dominate the timeline, not novel engineering.** Once the
  Makefile.vita and OS shim exist, most remaining work is the standard Vita-port cycle — build,
  resolve the next missing symbol, repeat — proportional to codebase size rather than to how hard
  the game's own logic is to understand.
- **Budget real GPU performance triage separately from the build-system work.** Getting a build to
  link and boot is mechanical; getting real-time 3D gameplay to hold a stable frame rate on Vita's
  GPU (see [03-vitagl/07-performance-best-practices.md](../03-vitagl/07-performance-best-practices.md))
  is a distinct, per-game effort that scales with how demanding the actual gameplay is (a racing
  game's static-ish tracks are a lighter lift than, say, a fighting game with several simultaneous
  characters and particle effects).
