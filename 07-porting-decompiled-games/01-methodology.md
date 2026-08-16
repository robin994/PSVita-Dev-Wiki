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
whole-game problem.** The decompiled game code above it — often several hundred thousand lines —
calls only libultraship/N64-SDK-shaped functions and needs zero changes for a new target.

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

### Correction (2026-08-16): there is likely no dedicated Vita platform shim at all

An earlier draft of this page claimed Ghostship's `lib/src/osViTable.c` was a Vita-specific OS/input
shim, written by analogy to a template. **That was wrong** — flagged directly by Rinnegatamante
himself, who ported Ghostship to Vita. `osViTable.c` is N64 Video Interface handling: genuine
decompiled/reimplemented N64 SDK code, shared across every platform this codebase targets, with
nothing Vita-specific about it. The claim came from a naive case-insensitive search for "vita" over
the repo's file list — `osViTable.c` matches that search purely because "Vi" + "Ta" spells "vita",
a coincidence, not a signal. Worth remembering as a cautionary example on its own: filename
substring matches are not verification.

**Second correction, same conversation:** the follow-up guess — that `gfx_opengl.h`/`gfx_sdl.h` are
generic and largely unmodified for Vita because no separately-named Vita backend file exists in the
tree — was also wrong, also flagged directly by Rinnegatamante. His fork's `gfx_opengl.cpp` is a
real, substantially modified file: it "contains a ton of optimizations" and doesn't use whatever
the upstream version calls "Prism," which the original does use. The methodology error this time
was different from the first: checking whether a *differently-named* file existed instead of
checking whether the *existing* file's content had been changed. Absence of a new filename is not
evidence of an unmodified file.

**What that changes:** the real Vita-specific engineering isn't in a small OS shim (correction #1)
and it isn't "basically free because the generic backend already works" (correction #2) — it's real,
nontrivial rendering-layer work already done once, living in Rinnegatamante's own `libultraship`
fork, reusable across every game built on that fork rather than something each new port reinvents.
That reframes where the actual effort in "porting to a new platform is a libultraship problem"
(above) goes: mostly into whichever fork/branch of libultraship already carries the Vita rendering
work, not into the target game's own tree at all.

**What this page does not know, and isn't going to guess a third time:** the specific content of
those `gfx_opengl.cpp` changes, what "Prism" is or why it doesn't apply to Vita, or how much of that
optimization work is generic-to-Vita versus tied to whichever game it was first written for. That's
a question for Rinnegatamante or for reading his actual fork, not something recoverable from public
repo browsing — don't have this page invent an answer next time either.

### LiveArea assets and packaging — nothing game-specific

Every reference port ships the same `livearea/` folder (`icon0.png`, a background image, a
`startup.png`, `template.xml`) and runs the same `vita-elf-create` → `vita-make-fself` →
`vita-pack-vpk` pipeline documented generically in
[02-vitasdk: homebrew app anatomy](../02-vitasdk/03-homebrew-app-anatomy.md). There is nothing about
this stage specific to being a decompiled game rather than any other homebrew.

## Building a from-scratch `Makefile.vita` against a live CMake project (SSB64, 2026-08-16)

The corrections above were about reading someone else's finished port. This section is different:
notes from actually writing a `Makefile.vita` for a game (BattleShip/SSB64) that had never been
Vita-ported before, working only from its `CMakeLists.txt` as a reference — no existing
`Makefile.vita` to copy. Went from "10000+ compiler errors" to a clean link (minus the known
`gfx_opengl.cpp`/Prism gap from the corrections above) by reading what CMake does for each error
and replicating it. The individual bugs are SSB64-specific; the *pattern* of bugs is not — expect
the same categories on the next game in this family.

- **A CMake project's per-target include-path split encodes a real architectural boundary — flatten
  it and things silently break in confusing ways.** This family's CMake typically gives the
  decompiled-C target its own `decomp/include`-style directory full of shims (`stdlib.h`, `stddef.h`,
  ...) that intentionally shadow the system headers, needed only because the *original* N64/IDO
  build had no real libc. If a flat Makefile puts that same `-I` on the C++ port/libultraship/Torch
  sources too, those shims shadow the *real* C++ standard library there — manifesting as bizarre,
  seemingly-unrelated STL errors (`wint_t`/`max_align_t` "not declared" deep inside `<cwctype>`/
  `<cstddef>`) with no obvious connection to the actual cause. Fix: GNU Make pattern-specific
  variables (`decomp/src/%.o: CPPFLAGS := -Idecomp/include ...`) to scope the shim include path (and
  any `-Wno-*` downgrades the decomp C needs) to exactly the same file set CMake scopes it to —
  never make it global.
- **Compiler *default* behavior drifts across GCC versions in ways that specifically break IDO-era
  decomp C.** GCC 14+ hard-errors `-Wimplicit-function-declaration` by default regardless of `-std=`;
  GCC 15's default C standard is C23, under which an empty-parens function pointer declarator
  (`int (*fn)()`, meaning "unspecified args" in K&R/C89 through C17) instead means "no args," which
  turns any decomp code assigning a differently-typed function pointer through such a declarator into
  a hard `-Wincompatible-pointer-types` error. Both are exactly the class of thing this codebase's own
  `CMakeLists.txt` already works around (`-Wno-implicit-int`, pinning `CMAKE_C_STANDARD 11`) — a
  from-scratch Makefile needs to *find and copy* that existing CMake workaround, not rediscover the
  problem from first principles. Grep the CMakeLists for `target_compile_options`/
  `CMAKE_C_STANDARD` on the decomp target before guessing.
- **VitaSDK is missing several POSIX headers that both the decomp's PC-port layer and libultraship
  itself assume unconditionally on "any real OS": `sys/mman.h` (mmap/mprotect — used for guard-paged
  coroutine stacks; fall back to plain `memalign`, accept no guard page), `sys/ucontext.h` (signal-
  handler register capture for crash backtraces/hang watchdogs — stub to no-ops, it's diagnostic-only),
  `dlfcn.h` (`dladdr` — used to distinguish "pointer inside a dynamically-loaded mod .so" from "inside
  the main binary"; Vita has no dlopen'd mods, so stub returns false/not-found). None of these are
  Vita bugs to work around cleverly — they're genuinely absent capabilities, and every other Vita
  homebrew in this ecosystem hits the same three.
- **Make does not track compiler-flag changes as a rebuild trigger — only source-file mtimes.**
  Iterating on a Makefile.vita by editing `CFLAGS`/`CPPFLAGS` repeatedly, without ever running `make
  clean`, silently leaves already-built `.o` files compiled against *stale* flags. This produced a
  real, confusing symptom here: a function whose `#ifdef PORT` guard was clearly satisfied still
  linked as "undefined," because its `.o` predated the Makefile edit that added `-DPORT=1`. Any time
  a "why is this still broken, the fix is right there" moment shows up, check the `.o`'s mtime against
  the Makefile's before spending more time on the guard logic itself — and run a full clean rebuild
  once before trusting a "zero errors" result as authoritative, especially right before the link step.
- **`file(GLOB_RECURSE ...)` vs `file(GLOB ...)` in the CMakeLists matters enormously when porting to
  Make.** `$(wildcard dir/*.c)` has no recursive form. A fighting game's decomp source is organized
  in deep per-character/per-stage subdirectories (`decomp/src/ft/ftchar/ftmario/`, ...); a flat
  Makefile source list silently misses all of it, and the resulting undefined-symbol list at link
  time is enormous and looks unrelated (hundreds of `ftCommon*`/`grXxxMakeGround`/`mnXxxStartScene`-
  shaped names). `$(shell find dir -type d)` fed into the same per-directory glob pattern fixes it in
  one line — cheaper than hand-enumerating every subdirectory once you notice the shape of the
  problem.
- **CMake's `FetchContent`-pulled dependencies (a pinned imgui tag + patch, a source-only library like
  `libgfxd` compiled directly into the target rather than linked) need to be vendored at the *exact*
  same version/patch for a hand-rolled Makefile, not "whatever's latest on GitHub."** Read the
  `GIT_TAG` and any `PATCH_COMMAND` in the `FetchContent_Declare` block and reproduce both exactly —
  a version drift here reintroduces bugs the pin/patch was specifically fixing.
- **A vendored third-party tool can misbehave when it shares a translation unit's global compile
  flags with the parent project, even without any include-path conflict.** `libgfxd`'s six microcode-
  variant source files each self-select their own N64 GBI dialect internally
  (`uc_f3d.c`: `#define F3D_GBI` before `#include "uc.c"`); the parent project's own
  `-DF3DEX_GBI_2=1` (needed by the *game's* code) leaking into libgfxd's compilation via shared
  `CFLAGS` made two mutually-exclusive microcode dialects visible in the same translation unit
  simultaneously — surfacing as a "duplicate case value" hard error nowhere near the actual cause.
  Fix was a targeted `-U` in that file's own pattern-specific `CPPFLAGS` to cancel just the one
  colliding macro, not a global flag change.
- **Desktop-only dev/debug/social features need per-symbol stubs, not just excluding their `.cpp`
  files — check every call site, not just the ones this family's existing Android port happens to
  gate.** RenderDoc capture triggers, Discord Rich Presence, a curl-based self-updater, a signal-
  backtrace hang watchdog, and third-party USB-HID controller-adapter support (`hidapi`) are all
  real features this family's desktop build ships that have no Vita equivalent. This codebase's own
  Android port already excludes several of these `.cpp` files (documented in its CMakeLists with a
  `CMAKE_SYSTEM_NAME STREQUAL "Android"` block) — a good starting list — but at least one call site
  (a per-frame Discord presence tick) turned out to be unconditional even on Android, meaning Android
  must be relying on something not visible from reading the CMakeLists alone. Trust the exclusion
  list as a starting point, but verify by actually trying to link, not by assuming parity with
  Android's guard.
- **Codegen steps CMake wires as `add_custom_command`/`add_custom_target` (credits-text encoding, a
  procedural PNG+C-array bake for a UI scroll arrow) need to be run once by hand for a Makefile that
  doesn't replicate the full custom-command dependency graph.** These show up as a plain "file not
  found" fatal error for a generated header with no `.c`/`.cpp` counterpart anywhere in the tree —
  the signal that it's generated, not missing, is that `git status`/a repo-wide `find` shows no
  source for it, but the CMakeLists has an `add_custom_command(OUTPUT <that exact path> ...)` block
  naming the exact Python tool that makes it.

## Best practices

- **Two wrong guesses in a row on the same question means stop guessing.** "It's this file" (filename
  match, no content read) and "it's basically nothing" (no differently-named file, existing file's
  content never diffed) were both wrong. The actual answer needed the author. Public repo browsing
  can tell you the build-system shape; it can't tell you what's been quietly rewritten inside a file
  that was already there upstream.
- **Don't fight CMake for this family of project.** The established convention across every shipped
  port in this family is a standalone `Makefile.vita`, not a VitaSDK CMake toolchain file — that's
  a deliberate, repeated choice across independent ports, not an oversight worth "fixing" by
  integrating CMake properly.
- **Treat the asset pipeline as already solved.** If the game already has a working Torch/YAML
  asset-extraction setup for desktop, a Vita port touches none of it — don't spend planning time
  here.
- **The reusable asset across games in this family is the author's own patched `libultraship` fork,
  not a template you rebuild per game.** Budget for obtaining/porting that fork's actual rendering
  work rather than assuming the generic backends just work once linked against vitaGL.
- **Budget real GPU performance triage separately from the build-system work.** Getting a build to
  link and boot is mechanical; getting real-time 3D gameplay to hold a stable frame rate on Vita's
  GPU (see [03-vitagl/07-performance-best-practices.md](../03-vitagl/07-performance-best-practices.md))
  is a distinct, per-game effort that scales with how demanding the actual gameplay is (a racing
  game's static-ish tracks are a lighter lift than, say, a fighting game with several simultaneous
  characters and particle effects).
- **When writing a `Makefile.vita` from scratch (no existing one to copy), the game's own
  `CMakeLists.txt` is the answer key, not just a source list.** Every category of "mysterious" build
  error in the section above had an exact, already-solved counterpart sitting in the CMakeLists —
  a `target_compile_options` block, a `FetchContent_Declare`'s `GIT_TAG`, a per-target
  `target_include_directories` split, an `add_custom_command`. Grep the CMakeLists for the failing
  symbol/flag/filename before treating the error as novel.
- **Run a full `make clean` before trusting a "zero build errors" result, at least once right before
  attempting the link stage.** Make does not invalidate object files when only compiler flags change
  (see above) — a long flag-iteration session can leave a "clean" build silently stale.
