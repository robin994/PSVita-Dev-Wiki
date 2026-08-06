# VitaSDK Best Practices — Consolidated Checklist

A summary of the recurring, high-value practices from across this section, gathered in one place.

## Memory

- Choose the right pool deliberately (general RAM / CDRAM / physically-contiguous / cached vs
  uncached) — don't default everything to general RAM. See
  [Hardware: memory architecture](../01-hardware/04-memory-architecture.md).
- Query actual free memory at runtime (`sceKernelGetFreeMemorySize`) rather than hardcoding assumed
  totals — usable memory varies by firmware version and active system features.
- For anything allocated through a callback-based API (media playback memory callbacks especially),
  **track what you allocate yourself** and explicitly release anything still outstanding after
  teardown, rather than trusting the SDK's own close path to reliably fire your deallocator on every
  exit route (error paths and cancellation paths are the ones most likely to skip it).
- Watch for resource-pool exhaustion as the root cause of "works once, fails the second time"
  symptoms — a classic leak-shaped bug pattern on this platform's small, pooled memory regions.

## Threading

- Size thread stacks deliberately — there's no lazy growth; undersizing produces corruption-shaped
  bugs, not clean crashes.
- Remember priority direction: lower number = higher priority.
- Use `sceKernelDelayThread` to pace polling loops instead of busy-waiting — CPU cores are a scarcer
  resource here than on a modern desktop.
- Join worker threads (`sceKernelWaitThreadEnd`) before touching state they own, rather than
  assuming completion.

## Build & linking

- Pin known-working versions/commits of fast-moving community graphics dependencies (vitaGL,
  imgui-vita) rather than always tracking latest — breaking API changes between versions are real
  and can produce a binary that compiles but doesn't render correctly.
- When you hit undefined-reference errors for an unfamiliar symbol prefix from a library you're
  already linking, check for a missing *optional backend* dependency (codec/compression libraries
  bundled as separate static libs) before assuming the library itself is broken.
- If a project pins an old CMake version, that's usually there for a reason (old `CMakeLists.txt`
  syntax/policy assumptions) — don't casually "modernize" past it without testing.

## Compatibility

- Treat rear touch, motion sensors, and camera as optional, queryable capabilities — PSTV lacks all
  three, and it's a real, actively-used deployment target for certain categories of homebrew.
- Load required sysmodules (`sceSysmoduleLoadModule`) explicitly before using their API surface —
  many aren't auto-loaded.
- Keep cross-referenced identifiers (title ID especially) in sync across separately-built binaries
  (main app + any companion plugin) manually — nothing in the toolchain enforces this for you.

## Verification

- Vita3K is a great fast-iteration tool but is not behaviorally identical to real hardware —
  verify hardware-adjacent behavior (media decode, precise timing, pooled-resource limits) on a real
  device before calling something done.
- Real-time on-device logging (`debugnet` or similar) is worth reaching for when investigating
  timing/ordering-sensitive bugs across threads or SDK callbacks — but treat it as temporary
  instrumentation to remove once the investigation concludes, not permanent shipped code.

## Reverse-engineered SDK surface

- The header/NID surface is community-reverse-engineered, not an official Sony release — treat
  unfamiliar constants (especially undocumented event-ID values in callback-based APIs) as
  convention to verify empirically, not guaranteed-correct documentation.
- When something behaves unexpectedly and you've ruled out your own code, "the assumed
  behavior/constant might just be wrong or incomplete" is a real, non-rare possibility worth
  checking rather than dismissing.
