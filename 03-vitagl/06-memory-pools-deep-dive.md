# vitaGL: Memory Pools Deep Dive

[Initialization & memory pools](02-initialization-memory-pools.md) introduces the four pools
(`VGL_MEM_RAM`, `VGL_MEM_VRAM`, `VGL_MEM_SLOW`, `VGL_MEM_BUDGET`) and how sizing works at init. This
page goes deeper on *choosing correctly between them* and *diagnosing pool-related bugs* — the part
that tends to matter most once an app is past initial bring-up.

## Choosing a pool for a resource

Think in terms of **who actually consumes this memory, and what does that consumer require**:

- **Ordinary GPU-rendered resources with no special requirement** (most textures, vertex/index
  buffers, render targets your own draw calls read) — `VGL_MEM_RAM` is the sensible default. It's
  the biggest pool and the one vitaGL itself defaults most allocations to.
- **Resources you specifically want off the CPU-contended general pool, dedicated to the GPU** —
  `VGL_MEM_VRAM` (CDRAM). Genuinely GPU-only resources can benefit from living here instead of
  competing with everything else for general RAM bandwidth/capacity, though for most homebrew this
  is a secondary optimization, not a correctness requirement the way the next pool is.
- **Anything a hardware DMA-capable block reads/writes directly without going through the GPU's own
  memory-management path** — `VGL_MEM_SLOW` (physically-contiguous) is the obvious first guess, and
  is genuinely correct for some consumers, but verify against real hardware rather than assuming:
  `SceAvPlayer`'s video-frame texture memory turned out to need neither `VGL_MEM_SLOW` nor even
  vitaGL's own `VGL_MEM_VRAM` sub-pool - both allocate successfully with no error, but the consumer
  (the hardware decoder) simply never produces output unless the memory came from a directly-
  allocated, GPU-mapped block instead of any vitaGL pool API at all (see
  [Hardware: multimedia hardware](../01-hardware/07-multimedia-hardware.md) for the specifics and
  how this was actually confirmed). Putting a DMA-dependent resource on the wrong pool doesn't
  necessarily fail loudly — it can compile, run, allocate without error, and only manifest as decode
  failures or silent no-ops that look unrelated to the actual root cause.
- **Anything that needs to coexist cleanly with a native `sceCommonDialog` overlay** —
  `VGL_MEM_BUDGET`, a smaller pool specifically carved out for this coexistence case.

## VGL_MEM_SLOW is small — treat it as a scarce, correctness-critical resource

Unlike `VGL_MEM_RAM` (which you can grow via the `ram_threshold` init parameter, trading off against
general system RAM) `VGL_MEM_SLOW`'s total size is essentially fixed at boot, from whatever the
system reports as available physically-contiguous memory — commonly in the tens-of-megabytes range,
not hundreds. Anything routed to this pool needs to be genuinely justified (a real DMA/physical-
contiguity requirement from whatever consumes it), and needs careful, explicit lifecycle management —
see the case study below.

## Case study: a real leak-shaped bug in a callback-based memory contract

A genuinely observed failure pattern, worth internalizing as a general lesson regardless of which
pool ends up being correct for a given resource: an API that hands you memory-callback hooks
(allocate/deallocate pairs, as `SceAvPlayer` does for video-frame memory — see
[Hardware: multimedia hardware](../01-hardware/07-multimedia-hardware.md)) does **not** guarantee its
own close/teardown path reliably calls your deallocate callback for every resource it ever asked
your allocate callback for. Concretely: closing a video player didn't always trigger the expected
deallocation callback for whatever buffer it was actively using at the moment of close — meaning that
allocation stayed "alive" from the allocator's bookkeeping perspective even though nothing was using
it anymore. Across repeated open/close cycles (reopening a video, switching content), each such
leaked allocation permanently reduced the pool's available space, until eventually a *new* video
open failed outright for lack of room — a failure that, investigated superficially, looks like "this
new content doesn't fit" or "the pool is too small," when the actual root cause is unrelated: prior
sessions' allocations were never properly released. (This held true across both memory-pool choices
tried for this consumer - it's a property of the callback contract itself, independent of which pool
backs the allocation.)

**The fix pattern, generalizable beyond this one specific API**: don't trust a callback-based
allocate/deallocate contract to be perfectly honored on every code path. Track every allocation your
own allocate-callback hands out (a small fixed-size array/list is usually enough — this is not a
high-frequency allocation pattern), remove an entry when the expected deallocate callback *does*
fire (the common, expected path), and after the API's own teardown/close call returns, **explicitly
sweep and force-free anything still outstanding** in your tracking structure — rather than assuming
the close call already handled it.

## Diagnosing pool exhaustion

- **`vglMemTotal(pool)` / `vglMemFree(pool)`** — query total/free space for a specific pool at
  runtime. The single most useful diagnostic for "why did this allocation fail" or "is my pool
  actually the size I expect" questions — check these *before* assuming a resource is "just too big"
  for the pool.
- **Symptom pattern to watch for**: "works the first time in a session, fails on the second attempt
  at the same operation" is a strong signal of exactly this kind of leak — not a content-specific or
  size-specific problem, since the *first* attempt succeeded with the identical content.
- **A useful negative-result diagnostic technique**: if you suspect a pool-sizing problem, deliberately
  grow the pool (where growable — `VGL_MEM_RAM` via `ram_threshold`) and retest. If the failure is
  byte-for-byte identical afterward despite confirming (via `vglMemTotal`) that the pool genuinely
  grew, that's strong evidence the *pool size* was never the real variable — the failure is
  happening for an unrelated reason (a leak, a hardcoded internal decoder buffer requirement, a
  different pool being the actual constraint, or something outside memory sizing entirely), and
  continuing to tune pool size further is very unlikely to be the fix.

## Practical checklist

- Choose pools based on what the *consumer* of a resource actually requires, not habit - and verify
  against real hardware rather than trusting that a plausible-sounding pool name (like
  "physically-contiguous") is automatically the correct one for a given consumer.
- Treat allocations handed out through any callback-based memory contract (an SDK's
  allocate/deallocate hooks) as needing explicit lifecycle tracking on your own side — don't lean
  solely on the third party's close/teardown path to call your deallocator reliably.
- Reach for `vglMemTotal`/`vglMemFree` early in any allocation-failure investigation, not as a last
  resort.
- If growing a pool doesn't change a failure's exact byte-for-byte signature, stop chasing pool
  size as the explanation and look elsewhere.
