# Case Study: Super Smash Bros. 64

**Status (2026-08-18): the external blocker below was resolved — Rinnegatamante's Vita-patched
`libultraship` fork was obtained and merged (`vita-rinne-merge` branch) — and the project has since
moved through an extended real-hardware bring-up phase.** The build links and boots on real
hardware; the current frontier is a series of real-hardware-only crashes (never reproduced on the
Vita3K emulator) in the early boot chain, several confirmed and fixed, one still open. See
[Real-hardware bring-up findings](#real-hardware-bring-up-findings-2026-08-18) below for what was
found — this is the first genuinely hands-on, on-device section of this case study; everything
above it is retained as the original planning-stage writeup.

**Status (2026-08-16, superseded above): build-system work done, one confirmed external blocker.** A
from-scratch `Makefile.vita` (no prior Vita port of this game existed to copy) now compiles and links
the entire codebase — decomp C, libultraship, Torch, imgui, libgfxd, ~800 object files — cleanly on
VitaSDK, with a single remaining gap: the OpenGL rendering backend (`gfx_opengl.cpp`) needs
Rinnegatamante's actual Vita-patched `libultraship` fork (see the `prism/processor.h` note below and
[the methodology page's corrections](01-methodology.md#correction-2026-08-16-there-is-likely-no-dedicated-vita-platform-shim-at-all)) —
not something obtainable from the public generic backend. Everything documented here as "roadmap" in
earlier drafts of this page has now actually been attempted; see
[the methodology page's build-flag lessons](01-methodology.md#building-a-from-scratch-makefilevita-against-a-live-cmake-project-ssb64-2026-08-16)
for the general, reusable-elsewhere findings from actually doing it.

The full analysis this page summarizes was written up as a standalone report; see
`ssb64-vita-port-strategy.html` for the complete version with a build-system comparison table and
a phased plan.

## What the project actually is

The upstream repo is `JRickey/BattleShip` — a name with nothing to do with the board game. It's a
PC port of **Super Smash Bros. 64**, built from a complete decompilation
(`ssb-decomp-re`), using the same libultraship + Torch stack as every other game in this family.
Code-naming a decompilation port after something unrelated is standard practice in this scene — the
same reason the Zelda 64 port in this wiki's other examples is called "Ghostship" and the Mario Kart
64 port is called "SpaghettiKart." No ROM or copyrighted asset ships in the repo or in any port;
the end user supplies their own legally-dumped ROM at first run, same as every decomp-based
homebrew already in the NeoVitaDB catalog.

## Why the methodology applies directly

The repo's own structure already matches the pattern in page 1 almost exactly: a `libultraship/`
submodule, a `Torch/` submodule, a `port/` directory for the C++ port layer, and `yamls/` for the
asset extraction config. The project's own documentation explicitly names **SpaghettiKart** and
**Starship** (a Star Fox 64 port) as its architectural reference ports — SpaghettiKart being not
just architecturally similar but Rinnegatamante's own already-completed Vita port, making it the
closest thing to a literal template available. (See page 1's corrections, plural: neither "it's an
OS shim" nor "the generic backends just work as-is" held up under direct questioning. The real
rendering work lives inside Rinnegatamante's own `libultraship` fork — confirmed by him to be
substantially modified, content otherwise unknown from here. That fork, not a Makefile template, is
the asset worth getting for this port.)

## v1 scope: 60 fps + widescreen, no netplay

Confirmed target: a working v1 at a stable 60 fps with widescreen, netplay explicitly deferred to
v2. Both v1 features already exist upstream — this is a config-and-tune job, not new engineering:

- **Widescreen** (`port/widescreen/`) is a CVar-gated system already handling viewport, projection,
  and HUD-anchoring for 16:9. Vita's 960×544 is near-16:9. v1 work: flip the CVar on, confirm the
  three affected planes (viewport / projection / HUD anchors) render correctly through vitaGL — not
  building widescreen support.
- **60 fps** (`port/interpolation/`) already runs game logic at a fixed, audio-locked 60 Hz tick;
  render interpolation only *fans out* to multiples (120/180/240) above that. v1 needs none of the
  fan-out machinery — target the system's own default single-tick-per-render path (`k=1`).
  **Correction (Rinnegatamante, direct): this is not an open risk.** This exact game already runs
  at 60 fps on real Vita hardware today, under full N64 emulation, on his own DaedalusX64-vitaGL.
  Emulation (interpreting/HLE-recompiling RSP microcode and CPU instructions at runtime) is
  categorically more expensive than a native decompiled build rendering directly through vitaGL —
  if the emulated path already clears 60 fps on this hardware, the native port has more headroom,
  not less. Treat GPU throughput as settled; the real work is using the rendering path correctly
  (see the `libultraship` fork note above), which is an engineering-correctness question, not a
  raw-performance one.
- **Netplay is out of scope for v1**, full stop — don't design around it yet. The project's rollback
  architecture doc is the right starting point when v2 picks it up.

## Recommended sequence

1. ✅ **Done.** Build and play the existing desktop port first — the only working baseline once
   things start rendering wrong on-device. Confirm widescreen + interpolation both work there with
   defaults.
2. ✅ **Done.** Fork; write a `Makefile.vita` from scratch against the game's own `CMakeLists.txt`
   (no existing Vita port of this specific game to template from — SpaghettiKart's shares the family
   pattern but not enough file-for-file detail to copy directly). See
   [the methodology page](01-methodology.md#building-a-from-scratch-makefilevita-against-a-live-cmake-project-ssb64-2026-08-16)
   for the reusable lessons from doing this the hard way.
3. **Blocked, needs external input.** Get Rinnegatamante's actual Vita-patched `libultraship` fork
   (or the equivalent diff) before writing any rendering-layer code — his `gfx_opengl.cpp` already
   carries real, non-trivial Vita optimizations that don't exist upstream. Don't attempt to rebuild
   this from the generic backend; confirmed to be the wrong assumption twice already in this planning
   pass. Confirmed again empirically: excluding `gfx_opengl.cpp` and every other file/library gap it
   surfaced still leaves the entire rest of the codebase (decomp C, libultraship, Torch, imgui,
   libgfxd) compiling and linking clean — this really is the only remaining gap, not one of several.
4. ✅ **Done except for step 3's gap.** Iterate to a link — standard Vita-port loop. Everything that
   isn't the OpenGL rendering backend now links.
5. LiveArea assets + `vita-elf-create`/`vita-make-fself`/`vita-pack-vpk` packaging.
6. On-device bring-up: boot, controller mapping, audio.
7. Enable the widescreen CVar; fix whatever doesn't map cleanly to 960×544.
8. Confirm frame time against the native rendering path (`libultraship` fork). Not expected to be a
   struggle — DaedalusX64-vitaGL already runs this game at 60 fps on the same hardware under full
   emulation, so this step is verification, not open-ended tuning.
9. **v2, not now:** wire a Vita `sceNet` transport into the existing rollback-netcode boundary.

## Real-hardware bring-up findings (2026-08-18)

Once the `libultraship` fork was obtained and merged, the build genuinely booted — and immediately
hit a series of crashes that **only reproduce on real hardware, never on Vita3K**. The general,
reusable mechanisms behind these now live in the generic reference pages they actually belong in
(so they're findable from *any* future Vita project, not just this one) — this section covers only
how they showed up specifically in this game's port, plus this project's own status. For the full
bug list and current status, see the project's own `docs/bugs/` rather than this page.

- **The coroutine-stack kernel-syscall constraint** — see
  [Kernel/core APIs: coroutines, fibers & manually-swapped stacks](../02-vitasdk/04-kernel-core-apis.md#coroutines-fibers--manually-swapped-stacks)
  and [the general methodology page's note on this affecting the whole port family](01-methodology.md#real-hardware-bring-up-two-constraints-affecting-this-whole-port-family)
  for the mechanism and the confirmed fix pattern. In this specific port, it hit `ResourceManager`'s
  mutex, `std::promise`/`future`'s condition_variable, and newlib's `malloc`/`memalign` locks, each
  independently, before the pre-warm pattern was applied to each.
- **The audio-heap-never-initialized bug** — see
  [the general methodology page](01-methodology.md#real-hardware-bring-up-two-constraints-affecting-this-whole-port-family)
  for why this affects the whole port family, not just this game. Fixed here by adding the missing
  `sSYAudioCurrentSettings = dSYAudioPublicSettings;` copy at the top of this port's own audio-init
  bridge function.
- **The zlib/NEON-related crash** — see
  [Prebuilt library gotchas](../02-vitasdk/14-prebuilt-library-gotchas.md#vitasdks-prebuilt-zlib-genuinely-compiles-with-real-arm-neon-instructions)
  for the confirmed NEON finding and what was tried against it. **Status in this project: still
  open.** One candidate ruled out directly here: the `SDLAudioP2` audio thread (a genuinely
  separate, concurrently-running real OS thread, confirmed present in every crash's thread list) was
  eliminated entirely as a real-hardware test, and the same crash still occurred — so it isn't
  (solely) a race with that specific thread. The vitaGL garbage-collector thread (also genuinely
  concurrent, not yet tested this way) remains an open candidate for this specific port.
