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
- **A `POST_BUILD` asset-staging step (a file copied next to the executable, not compiled into it)
  that a hand-rolled Makefile never replicates fails silently at runtime, not at build time — and
  the runtime failure can be a hang, not a crash, making it far harder to spot than the codegen-step
  case above.** SSB64's CMakeLists zips `libultraship/src/fast/shaders/` into `f3d.o2r` via a
  `GenerateF3DO2R` custom target and copies it next to the desktop binary in a `POST_BUILD`
  `add_custom_command`; `Makefile.vita`'s VPK-packaging rule never picked this up, so the bundled
  VPK simply had no `f3d.o2r`. The bootstrap `ResourceManager::Init()` call that loads it was written
  to tolerate a missing/empty archive (`allowEmptyPaths=true`, logs and continues) — so the app
  booted fine, reached rendering ("Window OK"), and kept going for several more log lines. The actual
  damage was silent: `ResourceManager::Init()` permanently pauses its `BS::thread_pool` if
  `IsLoaded()` is false at that exact call (a real upstream comment: *"Nothing ever unpauses the
  thread pool since nothing will ever try to load the archive again"* — an assumption this port's
  two-phase loading, bootstrap archive now / game archive later via a separate `AddArchive()`,
  quietly violates). The first real resource load after that hangs the main thread forever inside a
  `condition_variable::wait()` on the paused pool, with no exception thrown, no crash log, nothing —
  just a frozen app. Diagnosed by pulling a real-hardware coredump (`ux0:data/.../psp2core-*.psp2dmp`,
  gzip-compressed ARM `ET_CORE` ELF) and parsing it with a Python-3-ported copy of xerpi's
  `vita-parse-core` (the upstream tool targets Python 2 + `pyelftools==0.24`; on a modern
  `pyelftools` the fix is dropping the `elftools.common.py3compat` import, which no longer exists,
  and reimplementing `str2bytes`/`bytes2str`/byte-vs-`chr` handling locally — the note-parsing logic
  itself, `ELFFile.iter_segments()`/`iter_notes()`, is unchanged across versions). The coredump's
  crashed-thread stack (`pthread_cond_wait` → `std::condition_variable::wait` →
  `std::__atomic_futex_unsigned<...>::_M_load_when_equal`) plus a stop reason that matched none of
  the known CPU-exception codes (`0x30002`–`0x30004`) was the tell that this was a hang caught by an
  external dump trigger, not a memory-access crash — worth checking for on *any* "app just freezes,
  no error" report before assuming it's unfixable without more logging. **General lesson: after
  porting a CMake `POST_BUILD` step to a hand-rolled Makefile's packaging rule, diff the *complete*
  file list the CMake step stages against what the Makefile actually bundles — a codegen step with no
  compiled counterpart is easy to notice missing (build fails); a plain file-copy step is not (build
  and boot both succeed, and the failure mode may not even be a crash).**

- **`-Wl,-q,--allow-multiple-definition` (added to silence one legitimate duplicate-symbol case) turns
  every *other* accidental duplicate global symbol into a silent runtime data-corruption bug instead
  of a build failure.** SSB64 ships two source files that both define a global array under the same
  name for different game releases — `RelocFileTable.us.cpp` / `RelocFileTable.jp.cpp`, each defining
  `gRelocFileTable[]` at a different length (2132 vs 2107) selected by a compile-time
  `RELOC_FILE_COUNT` macro. CMakeLists.txt handles this explicitly: `list(FILTER SSB64_SRC_PORT
  EXCLUDE REGEX ".../RelocFileTable\\.(us|jp)\\.cpp$")` then adds back only the file matching the
  selected version, with a comment spelling out why. A hand-rolled Makefile's directory-wide glob for
  `port/resource/*.cpp` has no equivalent selection step and pulls in *both* — normally a hard
  "multiple definition of `gRelocFileTable`" link error, except `--allow-multiple-definition` was
  already on the link line (for an unrelated, legitimate case elsewhere), so the linker silently kept
  one definition and discarded the other with no diagnostic at all. The build succeeded, the app
  booted and rendered fine, and gameplay proceeded well into the first real scene load before failing
  — because `RELOC_FILE_COUNT` (2132, the US value) is a compile-time constant baked independently
  into *every other* translation unit that reads the table, while the actual backing array that won
  the link was the *shorter* JP one (2107 real entries): every index at or past 2107 read past the
  end of that array into zeroed memory and came back NULL, and the entries *below* 2107 silently held
  the wrong release's data (JP's mapping at a given numeric id, not US's) rather than erroring either.
  Root-caused by comparing a live process's actual runtime array contents (added a one-off boot-time
  log dumping `RELOC_FILE_COUNT` and `gRelocFileTable[i]` at a few sample indices, tested on the
  Vita3K emulator for a fast iteration loop) against the same indices read two other ways — `grep` on
  the generated `.cpp` source and `strings` on the final linked ELF's `.rodata` — which agreed with
  each other but *not* with the runtime log; that three-way mismatch (source ✓, static binary ✓,
  runtime ✗) was the tell that pointed at "wrong object file won the link," not a data-generation or
  logic bug. **General lesson: `--allow-multiple-definition` is a blunt instrument — it doesn't
  distinguish the one duplicate you're expecting from any other. After adding it, audit the CMakeLists
  for every `list(FILTER ... EXCLUDE REGEX ...)` / conditional-source-selection block near the port's
  own files (not just third-party vendored code) and replicate each one in the Makefile explicitly,
  since none of them will fail loudly anymore if missed.**

- **A vendored Vita rendering layer merged from another game's fork can leave that game's own
  hardcoded app name behind in a method that looks generic.** `Ship::Context::GetAppBundlePath()`
  (distinct from `GetAppDirectoryPath()`, which this port's Vita branch already correctly returns
  `"ux0:data/battleship"` for) was still returning Rinnegatamante's own `"ux0:data/ghostship"` — his
  game's directory, carried over unnoticed through the libultraship merge because the method compiles
  and links fine either way; nothing catches a *wrong but valid-looking* path at build time. The
  practical effect: every `RealAppBundlePath()`-based lookup (`PortLocateFile`'s second search
  candidate, `port.cpp`'s mods-dir probe, `first_run.cpp`'s Torch/config search candidates) silently
  missed on Vita and fell through to whatever fallback came next — for the `f3d.o2r` bootstrap shader
  archive specifically, that fallback was a bare relative `"./f3d.o2r"`, which real hardware's archive
  loader can't open (unlike the Vita3K emulator, which resolved it as a happy accident and proceeded
  fine) — leaving `ResourceManager::Init()`'s bootstrap archive unloaded and pausing the thread pool
  forever (the exact hang from the entry above), even *after* bundling `f3d.o2r` correctly into the
  VPK. The two bugs looked identical from the crash (same `condition_variable::wait` stack) — the
  second one only surfaced by testing the first fix on real hardware, since Vita3K's looser path
  handling masked it. **Fixed at the source**: `GetAppBundlePath()` now returns `"app0:"` (the
  read-only VPK-mounted partition — matches `ArchiveManager::Init()`'s existing
  `archive.starts_with("app0")` prefix-strip special case, which expects exactly this format).
  **General lesson: when vendoring a rendering/platform layer from a sibling game's fork, grep the
  merged files for the *other* game's own name/IDs/paths** (`ghostship`, its title ID, its bundled
  filenames) as a dedicated pass — these don't fail loudly, they just quietly point at the wrong
  place, and a method name alone (`GetAppBundlePath`) gives no hint it's still scoped to someone
  else's install.

- **`spdlog::async_logger::flush_on(level)` is not fire-and-forget — its `flush()` blocks the calling
  thread until the background worker drains its queue *and* the OS-level write completes — and a
  release-build default of `flush_on(info)` combined with slow removable media turns routine logging
  into a long, silent stall that looks exactly like a hang.** Two earlier bugs this session (a missing
  `f3d.o2r` bundle, a leftover hardcoded app path from a merged fork) each produced a genuine thread
  stuck forever in `condition_variable::wait`, diagnosed from a real-hardware `psp2dmp` coredump. After
  fixing both, a *third* coredump from the same "black screen, then a psp2dmp appears" symptom showed
  a completely different signature: the main thread caught mid-syscall inside
  `__sfvwrite_r → __swrite → _write_r → sceIoWrite` — not stuck forever, just doing a slow, blocking
  write. `Ship::Context::InitLogging()`'s release-build branch uses `spdlog::async_logger` (a
  background-thread logger, `async_overflow_policy::block`) with `flush_on(spdlog::level::info)`.
  `ArchiveManager::Init()` logs per-resource at info while indexing a multi-thousand-entry `.o2r`, and
  each of those calls forces a synchronous wait for the background worker to catch up and fsync to
  disk — fine on a desktop SSD or Vita3K's host-filesystem-backed storage (both fast enough that this
  is invisible), but real Vita hardware's microSD write/fsync latency compounds across potentially
  thousands of forced flushes into a stall long enough to look frozen, with nothing on screen to say
  otherwise. **Fixed two ways**: lowered the Vita build's flush level to `err` (routine info-level
  archive-loading spam no longer forces a synchronous wait — flush still happens immediately for
  anything that actually matters), and added a two-frame "Loading..." ImGui draw right after window
  init and before the archive-loading block, so a load that's still slow at least doesn't read as a
  black, possibly-crashed screen. **General lesson: a coredump's *stack shape alone* doesn't tell you
  "deadlock" vs "slow blocking syscall" — check whether the exact same crash address recurs identically
  across multiple captures (this session's real deadlocks did; this one's addresses drifted between
  captures, consistent with different points in a long but finite write) before assuming a hang is a
  logic bug rather than an I/O-bound stall, especially on removable storage.**

- **A coredump-parsing tool's "nearest known symbol" address resolution can be flatly wrong once
  the caller is on the wrong side of a runtime-vs-static-file address offset — verify with a second,
  ground-truth tool before trusting the narrative it builds.** After the two real deadlocks above were
  fixed, a fourth real-hardware `psp2dmp` showed a genuine CPU exception (not the same recurring hang
  signature) with `vita-parse-core`'s own printed disassembly correctly landing on `_kill_r`
  (newlib's `kill()`) — but its raw "STACK CONTENTS AROUND SP" dump, fed through the *same*
  address-to-symbol heuristic, produced what looked like a plausible `abort → raise → __assert_func →
  DockBuilderCopyWindowSettings → Gui::DrawMenu` chain. Feeding those exact addresses to
  `arm-vita-eabi-addr2line -e battleship.elf.unstripped.elf` **directly** returned unrelated string
  literals (`.LC209` in `CollisionFactory.cpp`) — because those addresses were the tool's *runtime*
  (coredump) addresses, and `addr2line` needs the *static ELF's own* vaddr numbering; the fix is
  re-deriving the static address from the offset the tool itself already prints
  (`"battleship.elf@1 + 0xNNNNNN"` → static addr = `0x81000000 + 0xNNNNNN` for this build's link base,
  confirmed by cross-checking against `objdump -d` output around the candidate address to see it land
  cleanly on a real instruction/function boundary, not mid-instruction garbage). Once corrected, the
  same stack values addr2line'd to the *exact same* chain the tool's own (correctly-offset) labels had
  suggested — so the tool wasn't lying, the mistake was mine, querying it with the wrong address space.
  **General lesson: when cross-checking a coredump tool's symbol labels with `addr2line`/`objdump`
  directly, use the *offset the tool already computed* against the static file's own load base, never
  the raw coredump address — and treat a "nearest symbol" match to a `.rodata` string-literal label
  (`.LC123`) as a strong signal the address fed in was wrong, not that the crash landed in the weeds.**
- **`ImGui::DockBuilderSetNodeSize()` asserts on a non-positive size (`IM_ASSERT(size.x > 0.0f && size.y
  > 0.0f)`), and the very first `Gui::StartDraw()`/`DrawMenu()` call in an app's life can race a
  main-viewport size that hasn't settled yet** — the assert fired here specifically because the
  boot-time "Loading..." screen added for the entry above forced that first call earlier than before
  (previously the first real frame happened deep enough into the game loop that window geometry had
  long since settled, so this was a latent bug nothing had ever triggered). Root-caused precisely by
  chasing the corrected addr2line chain from the entry above straight to
  `Gui.cpp:528: ImGui::DockBuilderSetNodeSize(dockId, ImVec2(viewport->Size.x, viewport->Size.y))` and
  `imgui.cpp:19760`'s assert. **Fixed at the call site, not by walking back the loading screen**: gated
  the one-time dock-node setup in `Gui::DrawMenu()` on `viewport->Size.x > 0.0f && viewport->Size.y >
  0.0f` in addition to the existing `!DockBuilderGetNode(dockId)` check — `DockBuilderGetNode` stays
  null until the block actually runs, so an early frame with an invalid size just retries next frame
  instead of crashing, which protects *any* future early-frame caller, not just this one.

- **Reading the actual assert-message string out of the coredump's `.rodata` beats guessing from
  symbol-adjacent addresses — do it as soon as an `abort()`/`__assert_func` chain shows up.** A fifth
  real-hardware `psp2dmp`, from the *same rebuilt VPK* that fixed the `DockBuilderSetNodeSize` assert
  above, hit the identical `abort → raise → __assert_func` shape one call earlier in the frame — but
  this time the naive nearest-symbol guess on a deeper stack slot ("`ImVector<ImGuiContextHook>::erase`
  ... called from `ImGui_ImplSDL2_UpdateMonitors`") turned out to be pure coincidence: `objdump -d`
  right before that "return address" showed unrelated font-atlas code, not a `bl __assert_func`, and a
  `grep` for `AddContextHook` across the whole tree found only its own definition — nothing ever
  populates `g.Hooks`, so that erase loop's body can never even execute. The reliable path instead was
  reading the raw string bytes at the `__assert_func` call's own `func`/`failedexpr` argument addresses
  directly out of the *static* ELF's `.rodata` (`objdump -s --start-address=... --stop-address=...`,
  using the same tool-computed static offset from the entry two above) — which spelled out the exact
  literal condition text: `g.IO.DisplaySize.x >= 0.0f && g.IO.DisplaySize.y >= 0.0f && "Invalid
  DisplaySize value!"`, inside `ImGui::ErrorCheckNewFrameSanityChecks()` — pinpointing the failing
  `IM_ASSERT` line in `imgui.cpp` with certainty a stack-address guess never could. Root cause:
  `ImGui_ImplSDL2_NewFrame()` assigns `io.DisplaySize` from `SDL_GetWindowSize()` unconditionally, and
  VitaSDK's SDL2 port can hit an internal error path (an `SDL_SetError()` call was visible nearby in
  the same stack dump) that leaves the output ints as whatever was already on the stack instead of
  writing real values — not necessarily zero, so even ImGui's own `>= 0.0f` guard doesn't catch it.
  **Fixed at the backend**: only update `io.DisplaySize`/`DisplayFramebufferScale` when both dimensions
  come back genuinely positive, same "skip and let it settle" pattern as the entry above, this time one
  level earlier (`ImGui_ImplSDL2_NewFrame()` runs before `Gui::DrawMenu()`'s dock setup even starts).
  **General lesson: don't stop at "the crash is inside `IM_ASSERT`" — the four string arguments to
  `__assert_func` (file, line, func, failedexpr) are sitting in that frame's own stack slots as
  `.rodata` pointers; dumping their actual text is cheap and removes all doubt about which of a
  function's several assert lines actually fired, versus inferring it from context.**

- **A "leave it untouched if invalid" guard isn't actually safe unless the untouched default is itself
  valid — check what the field's own constructor sets it to.** The `io.DisplaySize` fix above (only
  assign when `SDL_GetWindowSize()` returns positive values, otherwise leave `io.DisplaySize` alone)
  still crashed on the very next real-hardware test, with the *identical* assert. `ImGuiIO`'s own
  constructor (`imgui.cpp`'s `DisplaySize = ImVec2(-1.0f, -1.0f);`) initializes it to `(-1,-1)`
  specifically as ImGui's own "not yet configured by the application" sentinel — so "leave it
  untouched" on a first-ever failing call left that already-invalid negative default in place, and
  `ErrorCheckNewFrameSanityChecks()`'s `>= 0.0f` assert fired exactly as before. Confirmed via the same
  `.rodata`-string-extraction technique as the entry above: identical string addresses, identical
  assert text, on a coredump from the rebuilt VPK. **Fixed by falling back to `(0,0)` instead of
  leaving the stale value** — `0.0f >= 0.0f` passes the check, and a zero-sized display just means
  ImGui skips meaningful rendering for that one frame rather than crashing. **General lesson: a defensive
  "skip the update, keep the old value" guard is only as safe as whatever the old value could
  legitimately be — for a field with a deliberate invalid-by-default sentinel (common in APIs that want
  to detect "caller never configured this"), skipping the write doesn't skip the invalid state, it just
  preserves it. Check the type's own default/constructor before trusting a skip-on-failure guard.**

- **A recurring "stuck mid-`sceIoWrite`" coredump signature isn't always the slow-flush issue two
  entries above — confirm with the user whether it's actually slow (finite) or a genuine permanent
  hang before assuming the same root cause applies.** After the `flush_on(err)` fix, real-hardware
  testing progressed much further (into actual game-thread/coroutine creation) before hitting the
  *identical* `fflush → __sfvwrite_r → __swrite → sceIoWrite` stack shape again. Asked directly whether
  waiting longer (a minute or two) let it continue — it didn't; this was a genuine permanent hang, not
  slow disk I/O. `port_log()` (`port/port_log.c`) writes to a single shared `FILE*` with **zero
  synchronization**, and by this point in boot several real OS threads (audio, DMA/scheduler, the main
  thread) are all calling it concurrently — consistent with two threads racing VitaSDK newlib's
  internal per-`FILE` lock with no forward progress. **Fixed by wrapping every `port_log()` file
  operation in a `pthread_mutex_t`** — cheap, and correct regardless of whether newlib's own locking
  was actually the culprit, since unsynchronized multi-threaded writes to one `FILE*` are undefined
  behavior on any platform. **General lesson: the same crash signature recurring after a fix doesn't
  necessarily mean the fix failed — a quick "does waiting help?" check separates "still the same slow
  I/O" from "a different, newly-reachable bug that happens to look identical at the syscall level."**

- **When a coredump shows a thread stuck inside a `write()`/`fflush()` syscall repeatedly, at
  byte-identical register/stack values across independent runs, don't stop at "it's stuck" — dump the
  actual buffer contents from the coredump's own memory to see what was being written.** The
  `port_log()` mutex fix (entry above) didn't change anything: the *next* real-hardware test hit the
  identical `fflush → __sfvwrite_r → __swrite → sceIoWrite` stack, at the exact same stack address and
  file-buffer pointer, as before the fix — inconsistent with a timing-dependent race (those don't
  usually reproduce byte-for-byte). `vita-parse-core`'s `CoreParser.read_vaddr(addr, size)` reads raw
  bytes straight out of the coredump's own `PT_LOAD` segments by runtime virtual address (no static-ELF
  offset math needed here, unlike code addresses — this is *data* the process itself wrote, not
  something resolved against the static file); pointing it at the write's own buffer pointer register
  (`R1`) for the write's own size (`R2`, `0x400`/1024 bytes here) dumped the exact log text that was
  queued — which matched `ssb64.log`'s actual on-disk content byte-for-byte, confirming this was stdio's
  *very first* buffer-full flush of the freshly-`fopen(path, "w")`'d log file, always landing at the
  same point deep in boot (mid game-thread creation) because Vita homebrew has no ASLR and the exact
  same call sequence runs every launch. **Fixed by forcing that first flush to happen immediately in
  `port_log_init()`** (write one byte, `fflush()` immediately after `fopen()`) instead of letting it
  land wherever the buffer naturally fills on its own — extending a just-truncated file involves
  one-time FAT/cluster-allocation work on some memory cards that a scheduler-watchdog can trip over if
  it happens to land during a timing-sensitive window later in boot; paying that cost immediately, with
  nothing else running yet, means every later flush is an ordinary append instead. **General lesson:
  `CoreParser.read_vaddr()` turns "some thread is stuck writing something" into "here is the literal
  content it was writing" — much stronger evidence than register/backtrace analysis alone for I/O-stall
  bugs specifically (as opposed to code-address analysis, where the static-offset correction from
  entries above still applies).**

- **Two targeted mitigations in a row (mutex, force-first-flush-early) left the exact same coredump
  signature unchanged — byte-identical buffer contents, not just the same call stack — which was the
  signal to stop patching around the symptom and remove the game thread from the I/O path entirely.**
  Re-dumping the write's buffer contents after the "force first flush early" fix showed the *same*
  1024 bytes of accumulated log text as before, meaning stdio's second buffer-full flush still landed
  at the identical logical point — proof the stall was never about "first write to a freshly-truncated
  file," just genuinely slow synchronous disk I/O on this real hardware's memory card, reliably slow
  enough to trip whatever threshold the automatic-psp2dmp watchdog uses (stop reason `0x10006` has
  never once corresponded to a real CPU exception across any crash this port has hit — only ever to a
  thread parked inside a write syscall, consistent with an external "no scheduler progress" monitor
  rather than a hardware trap). **Fixed by moving all `port_log()` disk I/O off every calling thread
  entirely**: `port_log()` now only `vsnprintf`s into a stack buffer and pushes it onto a small
  fixed-size ring buffer (in-memory, no I/O, drops the line instead of blocking if the queue is ever
  full) guarded by a mutex + condvar; a single dedicated writer thread drains the queue and does the
  actual `fputs`/eventual `fflush`. Whatever the SD card's real per-write latency is, it can no longer
  make the *watched* thread block on it — a structural fix instead of a third attempt to dodge the
  specific timing window. **General lesson: when a fix that targets a specific hypothesis (locking,
  first-write cost) leaves the *exact* same evidence unchanged — not just the same symptom, but literally
  the same bytes in the same buffer — that's a sign the hypothesis was never the mechanism, and it's
  time to remove the suspect operation from the critical path rather than keep tuning around it.**

## Best practices

- **For on-device logging, emit through `sceClibPrintf` (`<psp2/kernel/clib.h>`) instead of
  `fprintf`/`fputs` to a file — but from exactly *one* dedicated thread, not from every caller.**
  `sceClibPrintf` goes straight through a `SceLibKernel` syscall — no newlib stdio buffering, no
  filesystem, no `fflush`/`write()` chain to get stuck in the way file logging repeatedly did on this
  project (see the `port_log()` entries above). It is **not**, however, free of blocking risk when
  called concurrently: the first real-hardware test after switching to it hit a *new* stall, with the
  main thread genuinely parked in a `SceLibKernel` wait call, the queued log line and the kernel UID it
  was waiting on both sitting on `port_log()`'s own stack frame in the coredump — landing exactly when
  several OS threads (audio, DMA/scheduler, controller) all spun up in a burst during coroutine boot
  and each called `sceClibPrintf` at once. Consistent with kernel-side serialization on the shared
  debug-output channel under concurrent callers — the same "many threads hit a slow shared I/O path at
  once" shape as the file-write stalls, just on a different resource. **Fixed by funneling every
  `sceClibPrintf` call through the same single dedicated writer thread already used for the async file
  queue**, instead of calling it directly from whichever thread produced the log line — removing the
  inter-thread contention entirely rather than trying to make concurrent calls to it safe. General
  pattern: format the line and enqueue it (fast, in-memory, no syscalls) from any caller; have exactly
  one thread dequeue and do *all* the actual I/O — both the `sceClibPrintf` call and the file write —
  serially. Never call `sceClibPrintf` directly from multiple threads without this funnel.
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
