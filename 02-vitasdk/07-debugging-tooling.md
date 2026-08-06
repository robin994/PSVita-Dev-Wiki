# Debugging & Tooling

There's no host-native build for most Vita homebrew (see
[Project setup & build system](02-project-setup-build-system.md)) — you're always cross-compiling
for the device, which shapes the whole debugging workflow around "get feedback from the actual
console" rather than a local desktop debug loop.

## Deploying builds to a device

**VitaShell** (see [Hardware: system software layers](../01-hardware/09-system-software-layers.md))
is the standard tool for getting a freshly built `.vpk` onto a real device — its built-in FTP server
lets you push files from your dev machine directly onto the console's storage over the local
network, which is the fastest iteration loop for "build, transfer, install, test" without physically
moving a memory card back and forth.

## Real-time on-device logging

Standard `printf`-to-a-terminal debugging doesn't exist in the usual sense — there's no attached
console the way a desktop app has stdout. Two complementary approaches cover most needs:

- **File-based dumps**: write diagnostic state to a file on `ux0:` (`sceIoOpen`/`sceIoWrite`) at
  key points, then retrieve it afterward via VitaShell/FTP. Simple, reliable, zero extra
  dependencies — the right default for "capture state at a specific failure point" debugging. The
  downside is it's inherently after-the-fact — you get a snapshot, not a live stream, so it's weaker
  for understanding *timing*/*ordering* of events across threads.
- **Real-time UDP logging** (e.g. the community `debugnet` library,
  `github.com/psxdev/debugnet`): the app calls `debugNetInit(pc_ip, port, level)` once at startup
  and `debugNetPrintf(level, fmt, ...)` anywhere you'd otherwise reach for a desktop `printf`; a
  listener on the dev machine (as simple as `socat udp-recv:<port> stdout`) prints each line live as
  it arrives. This is genuinely valuable for anything involving timing, ordering, or
  cross-thread/cross-callback interaction — e.g., understanding what order a sequence of SDK
  callbacks actually fires in on real hardware — in a way a post-hoc file dump can't show you as
  clearly. **Treat it as temporary instrumentation**: wire it in for an investigation, pull it back
  out (the library dependency, the init call, every call site) once you're done, rather than leaving
  live network logging permanently built into a shipped release.
- **`psp2shell`** is a similar, somewhat heavier-weight alternative some projects use instead of
  `debugnet` — same broad idea (live diagnostic output over the network to a PC-side listener), a
  bigger setup footprint in exchange for more built-in functionality (remote command execution,
  not just one-way logging).

## Emulator (Vita3K) vs real hardware

**Vita3K** is the community Vita emulator, genuinely useful for a fast local iteration loop (no
physical device round-trip needed for most testing), but it is **not behaviorally identical to real
hardware**, and treating "works in the emulator" as proof of correctness is a real, recurring source
of shipped bugs. The most concrete, well-documented gap: the emulator's software H.264 decoder does
not enforce the same profile/level ceiling the real hardware decoder does — content that plays fine
in Vita3K can fail outright on a real console (see
[Hardware: multimedia hardware](../01-hardware/07-multimedia-hardware.md)). More generally, any API
whose real hardware behavior comes from years of reverse-engineered, partially-documented
understanding (see [Overview & toolchain](01-overview-toolchain.md)) is a candidate for an emulator
that models the *documented* behavior faithfully while the *real* console does something subtly
different (an event ID that fires differently, a resource limit the emulator doesn't model at all).

**Practical rule**: use Vita3K for fast day-to-day iteration, but verify anything touching hardware-
adjacent behavior (media decode, precise timing, resource-limited subsystems like the
physically-contiguous memory pool) on a real device before considering it done — and if you only
have emulator access for a given change, say so explicitly rather than silently presenting
emulator-only verification as equivalent to real-hardware verification.

## Diagnosing "it works sometimes" / leak-shaped bugs

A useful general pattern for this platform, given the constrained/pooled memory model (see
[Hardware: memory architecture](../01-hardware/04-memory-architecture.md)): symptoms like "works the
first time, fails the second time" or "works at low settings/resolution but not high" often point at
a resource-pool exhaustion issue (a callback-based deallocation contract that isn't reliably
honored on every code path, accumulating leaked allocations across repeated use) rather than a
straightforward logic bug. When you see that shape of symptom, checking whether something is
tracking and force-releasing its own allocations on every exit path — rather than trusting an SDK
callback to always fire — is a productive early hypothesis.
