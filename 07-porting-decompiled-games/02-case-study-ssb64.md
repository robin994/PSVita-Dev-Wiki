# Case Study: Super Smash Bros. 64

**Status (2026-08-16): build-system work done, one confirmed external blocker.** A from-scratch
`Makefile.vita` (no prior Vita port of this game existed to copy) now compiles and links the entire
codebase — decomp C, libultraship, Torch, imgui, libgfxd, ~800 object files — cleanly on VitaSDK,
with a single remaining gap: the OpenGL rendering backend (`gfx_opengl.cpp`) needs
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
