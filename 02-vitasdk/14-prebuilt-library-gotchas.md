# Prebuilt third-party library gotchas

Most non-`sce*` libraries a Vita homebrew project links against (`zlib`, `libzip`, `libcurl`, `SDL2`,
and everything else `vdpm` installs — see
[Alternative toolchains & deployment](11-alternative-toolchains-deployment.md)) are prebuilt
binaries you didn't compile yourself and don't control the exact build flags for. Their public API
looks like ordinary, portable C — nothing about calling `inflate()` or `malloc()` *looks*
platform-specific — but the compiled code underneath can still make assumptions about the CPU that
don't hold on every ARM implementation, or interact with this platform's memory model in a way a
desktop build of the same library never would. This page collects confirmed, worked cases rather
than general "be careful" advice — add to it as new ones get found.

## VitaSDK's prebuilt zlib genuinely compiles with real ARM NEON instructions

**Confirmed directly via disassembly, not inferred from symptoms**: running
`arm-vita-eabi-objdump -d` on VitaSDK's `arm-vita-eabi/lib/libz.a` (zlib 1.3.2 at time of writing —
check `zlibVersion()`/the `deflate 1.x.y` string embedded in the archive for whatever version is
actually installed, since this can change with a `vdpm upgrade`) shows genuine `vld1`/`vst1` NEON
load/store instructions inside both `inflate` and `inflate_fast`. This matters because:

- **A real-hardware-only crash class was observed with PC landing directly inside `inflate_fast`'s
  own compiled code** (a genuine fault while zlib's own instructions executed — not a kernel
  syscall trampoline, and not misattributed disassembly; see
  [Debugging & tooling: emulator vs real hardware](07-debugging-tooling.md) for why Vita3K's
  software-emulated NEON path wouldn't necessarily reproduce whatever real-Cortex-A9 constraint this
  is), recurring non-deterministically across different archive entries and different points in a
  boot sequence between runs of an *identical* binary.
- **Two different fixes were tried against this, with different, informative results.** A
  buffer-size padding fix (giving `zip_fread`'s destination extra slack past the exact requested
  size, since a decompressor's fast path can plausibly write in wider bursts than requested)
  confirmed-fixed one specific, tiny (5-byte) archive entry's crash outright — but did not fully
  resolve the broader, recurring class of crash for other entries. A 16-byte-alignment fix (matching
  NEON's quad-word access width, applied via `memalign` instead of a default-aligned
  `std::vector`/`new[]` allocation) was tried on a much larger (~12 MB) buffer and caused a *worse*
  regression — a data abort from a zeroed vtable, consistent with that large `memalign()` call
  overlapping already-live heap memory, reverted rather than debugged further given the severity.
- **Neither fix fully explains the whole crash class** — as of this writing, the underlying issue
  remains open, and the crash's non-deterministic, cross-unrelated-code location (window init,
  archive loading, plain `malloc`, zlib itself, between runs of identical code — see
  [Kernel/core APIs: coroutines, fibers & manually-swapped stacks](04-kernel-core-apis.md#coroutines-fibers--manually-swapped-stacks)
  for a separate, confirmed contributing factor in the specific project this was found in) is more
  consistent with an earlier, broader memory-corruption source than with zlib/NEON alignment being
  the *whole* story on its own.

**The generalizable lesson, independent of whether alignment turns out to be the full explanation
here**: if you're feeding a prebuilt library's hot-path/decompression/checksum function a
heap-allocated buffer via a generic C++ container's default allocation, and you hit a real-
hardware-only crash with the fault PC landing *inside that library's own compiled code* (not a
kernel call, not your own code) — check what SIMD instruction set the actual installed build was
compiled with via `objdump`, don't assume "it's just C, alignment doesn't matter here." Confirm the
instruction set is really in use with an actual disassembly check before spending time on an
alignment-based fix; a `vdpm upgrade` can also change the installed library's version (and
potentially its build flags) out from under a project without anyone touching application code, so
re-verify rather than assuming a past finding about "the installed zlib" still holds after an SDK
update.

## Sourcing note

Everything on this page should be a *confirmed* finding (objdump output checked directly, a fix
tested on real hardware with an observed before/after result) — not a general worry about prebuilt
libraries in the abstract. If you're adding to this page, hold it to that same bar.
