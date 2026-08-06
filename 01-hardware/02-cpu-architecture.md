# CPU Architecture

## ARM Cortex-A9, quad-core

The Vita's CPU cluster is four ARM Cortex-A9 cores (ARMv7-A instruction set, 32-bit, in-order-issue
superscalar with out-of-order completion), each with a **NEON** SIMD unit and hardware VFP
(floating point). There's no 64-bit mode anywhere in this picture — all Vita homebrew is compiled
as 32-bit ARM (or Thumb-2) code.

For scheduling purposes, homebrew generally treats the CPU as offering **3 usable cores** for a
foreground application; one core is effectively reserved by the system. This isn't a hard
documented rule so much as the practical ceiling most homebrew projects converge on — spawning
more worker threads than that tends to just contend with system threads rather than add real
throughput.

## NEON

NEON is the SIMD extension — 128-bit vector registers, useful for the same things it's useful for
anywhere else (audio/video processing, math-heavy loops, batch coordinate transforms). VitaSDK's
toolchain supports it (`-mfpu=neon` and friends at the GCC level), but most homebrew doesn't hand-
write NEON intrinsics; it's more commonly encountered indirectly, through libraries that already use
it internally (codec libraries, some math libraries) than written from scratch in application code.
If you *are* chasing CPU-bound performance on this platform, NEON is one of the first places to
look before assuming you need to drop work onto the GPU instead.

## Threading model

Vita homebrew doesn't use POSIX threads directly at the syscall level — the underlying primitive is
`sceKernelCreateThread` and friends (`SceKernelThreadMgr`), with a real preemptive scheduler and
priority-based scheduling (lower numeric priority value = higher actual priority, matching most
other Sony/embedded RTOS conventions — this trips people coming from POSIX `nice`, where the
direction is the same, but from Win32 thread priorities, where it's often described the opposite
way). VitaSDK's newlib-based libc layers a `pthread`-compatible API over this for portability, so
most portable C/C++ code (including things like SDL2's threading) works unmodified — but be aware
that under the hood every "pthread" is a `sceKernelThreadMgr` thread with all that entails
(fixed-size stacks you specify up front, no lazy stack growth, kernel-visible thread names useful
for debugging).

**Practical implications:**
- Thread stack sizes are fixed at creation time (`sceKernelCreateThread`'s `stackSize` argument) —
  size them deliberately; there's no guard-page-triggered growth like a desktop OS gives you.
- Thread priority matters more here than on a desktop OS with dozens of cores to hide scheduling
  mistakes behind. A background thread accidentally created at too high a priority can visibly
  stall the render thread.
- `sceKernelDelayThread` (microsecond-granularity sleep) is the idiomatic way to yield/pace a
  thread loop; busy-waiting without it burns a full core doing nothing.

## Synchronization primitives

Below the pthread-compatible surface, the native primitives are mutexes (`sceKernelCreateMutex`),
semaphores (`sceKernelCreateSema`), event flags (`sceKernelCreateEventFlag` — useful for
one-to-many "wake up on any of these conditions" signaling, less common on desktop platforms but
idiomatic here), and condition variables. Homebrew that needs to coordinate a decode/render thread
pair (very common pattern — see the multimedia and vitaGL sections) typically reaches for these
directly rather than going through the pthread wrapper, since the native APIs expose a few Vita-
specific behaviors (like event flags) that don't have a clean POSIX equivalent.

## Cache and memory ordering

Cortex-A9 has separate L1 instruction/data caches per core and a shared L2 cache for the cluster.
Code that DMAs data or shares buffers between the CPU and GPU/video-decode hardware needs to be
conscious of cache coherency — this is one of the reasons some memory pools are allocated as
**uncached** (see [Memory architecture](04-memory-architecture.md)): for buffers the GPU or a
hardware decoder reads/writes directly, going through the cache would require explicit
flush/invalidate calls the SDK mostly hides from you by steering such buffers to uncached memory
instead.
