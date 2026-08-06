# Kernel / Core APIs

These are the "operating-system-level" APIs every non-trivial Vita app touches regardless of what
higher-level libraries (graphics, audio, UI) sit on top: threading, synchronization, memory
allocation, and file I/O. All exposed as `sceKernel*` / `sceIo*` user-mode syscall wrappers.

## Threading

- **`sceKernelCreateThread(name, entry, initPriority, stackSize, attr, cpuAffinityMask, optParam)`**
  — creates a thread. Note the **explicit, fixed stack size** — there's no lazy stack growth the way
  a desktop OS thread typically gets; undersizing this produces a stack overflow that can manifest
  as bizarre, hard-to-diagnose memory corruption rather than a clean crash, so err generous for any
  thread doing non-trivial recursion or large local buffers.
- **`sceKernelStartThread`** / **`sceKernelWaitThreadEnd`** — start a created thread and block until
  it exits, respectively. A common, correct pattern for any producer/worker thread whose result the
  caller needs before proceeding (a download thread, a decode thread) is: create, start, do other
  work, then `WaitThreadEnd` before touching shared state the worker thread owns — reusing
  worker-thread-owned globals without this join is a real, easy-to-introduce race condition.
- **Priority**: lower numeric value = higher scheduling priority (matches POSIX `nice` direction,
  opposite of how Win32 thread-priority levels are sometimes described colloquially). Getting this
  backwards is an easy mistake if you're translating intuition from the wrong prior platform.
- VitaSDK's newlib layer provides a `pthread`-compatible wrapper for portable code (SDL2's threading,
  for instance, rides on this) — under the hood it's still `sceKernelThreadMgr` threads with all the
  properties above.

## Synchronization

- **Mutexes** (`sceKernelCreateMutex`/`Lock`/`Unlock`) and **semaphores**
  (`sceKernelCreateSema`/`SignalSema`/`WaitSema`) — the usual suspects, behave as expected.
- **Event flags** (`sceKernelCreateEventFlag`/`SetEventFlag`/`WaitEventFlag`) — a bitmask-based
  signaling primitive letting a waiter block on "any of these bits" or "all of these bits" becoming
  set, useful for one-to-many wake-up patterns (a UI thread waiting on "either input arrived or a
  background task finished, whichever comes first") that don't map as cleanly onto a plain condition
  variable. Less familiar to programmers coming purely from a POSIX/pthreads background, but
  genuinely useful once you know it's there.
- **`sceKernelDelayThread(microseconds)`** — the idiomatic sleep/yield call for pacing a polling
  loop; prefer it over busy-waiting, which burns a full CPU core (a real resource on a 3-usable-core
  system — see [Hardware: CPU architecture](../01-hardware/02-cpu-architecture.md)) for no benefit.

## Memory allocation

- **`sceKernelAllocMemBlock(name, type, size, optParam)`** — the low-level allocator for
  pool-specific memory (general RAM, CDRAM/VRAM, physically-contiguous, cached vs uncached
  variants — see [Hardware: memory architecture](../01-hardware/04-memory-architecture.md) for the
  full pool breakdown and why the choice of `type` matters, not just the size).
- **`sceKernelGetFreeMemorySize`** — reports current free space per pool
  (`SceKernelFreeMemorySizeInfo`: `size_user`, `size_cdram`, `size_phycont`, ...). Query this at
  runtime rather than hardcoding assumed totals, since usable memory varies by firmware version and
  active system features.
- Ordinary heap allocation for general-purpose data (not tied to a specific hardware pool) typically
  just goes through standard C `malloc`/`free` via newlib, which itself is backed by a
  `sceKernelAllocMemBlock`-managed heap under the hood — you don't normally need to think about this
  layering unless you're deliberately routing something to a non-default pool.

## File I/O

- **`sceIoOpen`/`sceIoRead`/`sceIoWrite`/`sceIoClose`** — POSIX-*flavored* (not identical) file
  handles; standard C `fopen`/`fread`/etc. via newlib map onto these for portable code.
- **`sceIoGetstat`** (fills `SceIoStat`: mode, size, timestamps) — the idiomatic "does this exist /
  what are its properties" check; there's no lightweight separate `access()`-equivalent most
  homebrew reaches for instead.
- **`sceIoMkdir`/`sceIoRemove`/`sceIoDopen`+`sceIoDread`** — directory creation, file removal,
  directory listing.
- Always use full mount-point-prefixed paths (`"ux0:data/MyApp/..."`) — see
  [Hardware: storage & filesystem](../01-hardware/05-storage-filesystem.md) for what each mount
  point actually is.

## Module loading

- **`sceSysmoduleLoadModule(SCE_SYSMODULE_*)`** — many system libraries aren't loaded by default at
  process start and must be explicitly loaded before use (common examples: `SCE_SYSMODULE_AVPLAYER`
  for media playback, `SCE_SYSMODULE_NET` for networking, `SCE_SYSMODULE_RAZOR_CAPTURE` for
  profiling-capture support). Forgetting to load a required sysmodule before calling into its API
  surface is a common early-development mistake, and the resulting failure mode isn't always an
  obviously-labeled "module not loaded" error — sometimes it's just the API call failing in a less
  obvious way, so if a whole subsystem's API calls are failing right from the first call, "did I
  load its sysmodule" is a very early thing worth checking.
- **`sceKernelLoadStartModule`** — for loading actual `.suprx`/`.skprx` module files (companion
  plugins/drivers bundled with an app, or third-party modules) rather than a built-in system
  library by enum ID — see [Kernel plugins & taiHEN](06-kernel-plugins-taihen.md).
