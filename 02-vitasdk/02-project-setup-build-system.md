# Project Setup & Build System

## CMake is the standard

Virtually all VitaSDK projects build with CMake, using the toolchain file VitaSDK ships:

```sh
mkdir build && cd build
cmake -DCMAKE_TOOLCHAIN_FILE="$VITASDK/share/vita.toolchain.cmake" ..
make
```

A project's own `CMakeLists.txt` typically starts:

```cmake
cmake_minimum_required(VERSION 3.x)   # some older projects still pin 2.8-era minimums
if(NOT DEFINED CMAKE_TOOLCHAIN_FILE)
    set(CMAKE_TOOLCHAIN_FILE "$ENV{VITASDK}/share/vita.toolchain.cmake" CACHE PATH "toolchain file")
endif()
project(MyApp C CXX)
include("${VITASDK}/share/vita.cmake" REQUIRED)
```

`vita.cmake` supplies the Vita-specific build macros used below, on top of otherwise-ordinary CMake
`add_executable`/`target_link_libraries` usage.

## The Vita-specific build macros

- **`vita_create_self(output.self target_name [UNSAFE] [CONFIG exports.yml])`** — converts a
  standard ELF executable into a Vita SELF (signed ELF) file — `eboot.bin` for the main app, or a
  `.suprx`/`.skprx` for a plugin/kernel module. `UNSAFE` produces an unsigned/homebrew-appropriate
  SELF rather than requiring an actual Sony signing key (which homebrew obviously doesn't have)
  — CFW's loader accepts these. The optional `CONFIG` YAML file declares what a plugin/module
  *exports* for other modules to import (relevant for kernel modules and user-mode plugins — see
  [Kernel plugins & taiHEN](06-kernel-plugins-taihen.md); a plain standalone app usually doesn't
  need one).
- **`vita_create_vpk(output.vpk TITLEID eboot.bin VERSION ... NAME ... FILE src dest ...)`** —
  packages the final installable `.vpk`: the produced `eboot.bin`, a generated `param.sfo`
  (metadata: title ID, version, app name), and every bundled data file (icons, fonts, LiveArea
  assets, any `.suprx`/`.skprx` companions) at their target in-package paths. See
  [Homebrew app anatomy](03-homebrew-app-anatomy.md) for what actually goes into a VPK and why.

## Compiler/linker flags you'll commonly see

- `-Wl,-q` — generates relocatable output the Vita SELF-conversion tooling needs (a fairly universal
  flag across VitaSDK projects, not optional in most setups).
- `-Wl,--allow-multiple-definition` — sometimes needed when multiple static libraries in the
  dependency chain define overlapping weak symbols; a pragmatic workaround more than a "correct"
  fix, but common enough in this ecosystem's prebuilt library ecosystem to see routinely.
  `-Wl,--whole-archive <lib> -Wl,--no-whole-archive` around a specific library forces the linker to
  pull in *all* of that archive's object files rather than only the ones referenced so far — used
  when a library relies on static-initializer registration patterns (common for `pthread`-shaped
  libraries and for certain plugin-registration systems) that a normal "only link what's
  referenced" linker pass would otherwise drop.
- `-fpermissive` (C++) — some VitaSDK headers/typedefs don't line up perfectly with strict C++ type
  rules (a common friction point given the header set's reverse-engineered origin); a fair number of
  real-world VitaSDK C++ projects build with this to avoid hard errors on such mismatches. Worth
  treating pragmatically rather than as a sign your code is wrong — but also worth periodically
  checking what specific warnings it's silencing, since some really do indicate a genuine type
  mismatch (e.g., a callback signature not matching the SDK struct's expected function-pointer type
  exactly) that happens to still work at the ABI level on this platform but isn't strictly correct.

## Common linking gotchas

**Missing optional codec/compression backends.** Prebuilt VitaSDK ports of libraries like
`libcurl` or `SDL2_mixer` are frequently built with several *optional* backend libraries as
separate static dependencies that aren't linked automatically — undefined-reference errors for
symbols prefixed `op_*` (opusfile), `xmp_*` (libxmp tracker-module support), `ZSTD_*`, or similar
almost always mean you need to add that specific backend (`opusfile`, `opus`, `xmp`, `zstd`, ...) to
your own `target_link_libraries`, not that the base library build is broken.

**Vendored dependency version drift.** Fast-moving community libraries commonly used in graphics-
heavy homebrew — **vitaGL** and **imgui-vita** especially (see their respective sections) — have
had real, breaking API changes over their lifetime (a canonical example: `vglDrawObjects`'s argument
count changed between versions). If a project's source was written against one version and your
local checkout/prebuilt copy is a different one, you can get either a hard compile error (best case
— easy to spot) or, worse, code that *compiles* against a signature-compatible-but-behaviorally-
different version and produces a binary that builds fine but doesn't actually render on real
hardware. When pulling in a community graphics dependency, pin to a known-working commit/version
rather than always tracking the latest `HEAD`, and if something builds but doesn't render correctly
on-device, checking for a version mismatch against what the source was actually written for is a
worthwhile early diagnostic step.

**CMake version sensitivity in older projects.** Some longer-lived VitaSDK projects have
`CMakeLists.txt` files written against very old CMake minimums (occasionally as old as 2.8-era
syntax/policy behavior) that don't configure cleanly under a modern CMake without policy
adjustments — and forcing a newer policy version onto old syntax can silently break assumptions the
original `CMakeLists.txt` relied on (a classic example: a `CMP0046`-era policy flip changing how
forward-references to not-yet-defined targets in `add_dependencies` are treated). If a project
documents a specific pinned CMake version to use, that's usually there because someone already hit
this and worked around it by pinning rather than modernizing the build file — don't assume it's
just outdated advice.

## Emulator vs real hardware in your build/test loop

There is no host-native build target for most Vita graphics/system-API-heavy homebrew — you cross-
compile for the device every time, and "does it build" is a necessary but *not sufficient* signal of
correctness. Several real, non-hypothetical behavior gaps exist between **Vita3K** (the community
emulator) and real hardware — see
[Multimedia hardware: H.264 limits](../01-hardware/07-multimedia-hardware.md) for one concrete,
well-documented example (hardware video decoder level/profile enforcement) — so plan to verify
meaningfully complex changes on real hardware before considering them actually done, not just "it
built" or "it ran in the emulator."
