# Memory Architecture

This is the single most important page in the hardware section for anyone about to actually write
code — more Vita-specific bugs trace back to a misunderstanding of memory pools than to any other
single hardware quirk.

## The pools

The Vita does not present a single flat heap the way a desktop OS does. Distinct physical memory
regions exist, each with different properties, and the kernel API makes you choose explicitly which
one a given allocation comes from:

- **General user RAM (`USER_RW`)** — of the 512 MB main-memory die, a 365 MiB partition. Cached,
  general-purpose, allocated via `sceKernelAllocMemBlock` with types like
  `SCE_KERNEL_MEMBLOCK_TYPE_USER_RW`. This is where your heap, most textures (unless you route them
  to CDRAM), audio buffers, and general application data live. **The 365 MiB figure is not what you
  get by default**, though — see "Your app's actual RAM budget" below; without opting in, an app is
  capped at 256 MiB of this partition regardless of how much is physically present. Exact free-space
  figures still vary by firmware/active system features — query at runtime via
  `sceKernelGetFreeMemorySize` rather than hardcoding a number.
- **CDRAM / VRAM** — of the 128 MB video-memory die, a 112 MiB partition usable by applications
  (`SCE_KERNEL_MEMBLOCK_TYPE_USER_CDRAM_RW`); the remaining 16 MiB is reserved by the system.
  Graphics libraries (vitaGL included) commonly split their own texture/vertex-buffer allocations
  between general RAM and CDRAM.
- **Physically-contiguous memory ("phycont")** — a 26 MiB partition of main memory, **1 MiB-aligned**
  (every allocation from it effectively rounds up to a 1 MiB boundary, so many small allocations
  waste more of this pool than their nominal size suggests), guaranteed to be one unbroken run of
  physical addresses — required by hardware blocks that do DMA without an MMU/scatter-gather layer in
  front of them, most notably the **hardware video decoder** used by `SceAvPlayer`. Allocated with
  `SCE_KERNEL_MEMBLOCK_TYPE_USER_MAIN_PHYCONT_RW` (or the `_NC_RW` uncached variant). 26 MiB total,
  shared by everything in the app that uses it, is small enough to be a recurring source of real
  bugs — see the callout below. (Query the live figure via `sceKernelGetFreeMemorySize` rather than
  hardcoding 26 MiB — it's derived from what's actually available at boot, not a fixed constant.)
- **CDIALOG memory** — a 9 MiB partition (~8.77 MiB actually usable, after 1024-byte alignment)
  reserved for Sony's native common-dialog UI (on-screen keyboard, save-data dialogs, etc., via
  `sceCommonDialog`) when one is active; homebrew graphics layers sometimes carve out a small budget
  here too so a native dialog can coexist with the app's own rendering without fighting over CDRAM.
- **Reserved** — the remaining 112 MiB of main memory is reserved by the OS itself (shared modules,
  kernel, shell) and not available to applications under any flag.

## Your app's actual RAM budget: the `ATTRIBUTE2` param.sfo flag

`USER_RW`'s 365 MiB partition is the *ceiling*, not what an app gets by default. Without opting in, a
homebrew app is limited to a **256 MiB** budget (to leave headroom for the system to keep other apps
resident for fast-switching). To raise it, set `ATTRIBUTE2` in the app's `param.sfo` — in a CMake
project, via `VITA_MKSFOEX_FLAGS`:

```cmake
set(VITA_MKSFOEX_FLAGS "-d ATTRIBUTE2=12")
```

| Value | Extra RAM | Effective budget |
| --- | --- | --- |
| `4` | +29 MiB | ~285 MiB |
| `8` | +77 MiB | ~333 MiB |
| `12` | +109 MiB | ~365 MiB (the full `USER_RW` partition) |

`12` is the value most commonly used in the homebrew ecosystem. There's a real tradeoff: a bigger
budget claims more memory the system would otherwise keep other suspended apps resident with, at the
benefit of headroom for the app itself — worth setting deliberately based on the app's actual memory
needs, not reflexively maxed out on every project. See
[Homebrew app anatomy](../02-vitasdk/03-homebrew-app-anatomy.md) for the rest of what `param.sfo`
controls.

vitaGL exposes this pool split directly through its own allocation-type enum (`VGL_MEM_RAM`,
`VGL_MEM_VRAM`, `VGL_MEM_SLOW` for phycont, `VGL_MEM_BUDGET` for the common-dialog pool) — see
[vitaGL: memory pools deep dive](../03-vitagl/06-memory-pools-deep-dive.md).

## The physically-contiguous pool is small — and easy to exhaust silently

This deserves its own callout because it's a genuinely common real-world bug pattern. Anything that
routes memory through the phycont pool — most importantly, `SceAvPlayer`'s texture-memory callback
for video decode buffers — is drawing from a pool that might only be ~25-30 MB *total*, shared by
whatever else in the app also uses it. Unlike general RAM, there's no generous slack: a single
H.264 decode session can plausibly need 10-30 MB of reference-frame/output-buffer memory depending
on resolution and profile, and once that pool is exhausted, further allocations from it simply
fail — often silently, if the calling code (or a library like AVPlayer) doesn't surface the failure
clearly. Symptoms of phycont exhaustion tend to look like "it worked once but not the second time"
(if a previous session's allocation from this pool wasn't properly released) or "it plays fine at
low resolution but not high resolution" (bigger decode buffers).

**Practical rule:** if you're touching video decode, or anything else that allocates from a
physically-contiguous pool, always explicitly track and release what you allocate rather than
trusting a callback-based API to always call your deallocator reliably on every code path (close,
error, cancel). This exact failure mode — a close/teardown path that doesn't reliably fire the
expected deallocation callback, silently leaking phycont memory across repeated open/close cycles —
is a real, observed bug class on this platform, not a hypothetical.

## Uncached memory

Some memory block types come in cached and uncached (`_NC_`) variants. Uncached memory is used for
buffers a hardware block (GPU, video decoder, audio DMA) reads or writes directly without the CPU's
cache being kept coherent with it automatically — using cached memory for such a buffer without
explicit flush/invalidate calls around every hardware access is a classic source of Heisenbugs
(works most of the time, occasionally reads stale data) that's hard to reproduce and debug from C
code that "looks correct." When in doubt about a buffer's cache behavior, check what memory block
type the library or your own allocation actually requested.

## Querying available memory at runtime

`sceKernelGetFreeMemorySize` (filling a `SceKernelFreeMemorySizeInfo` struct) reports free space
per-pool at the moment it's called — `size_user` (general RAM), `size_cdram`, `size_phycont`, and a
couple of others. This is the correct way to make any decision that depends on "how much room do I
actually have," rather than hardcoding assumed totals — both because firmware versions and active
system features shift the numbers, and because a pool's *total* size and its *currently free*
amount are two different questions (the total for the physically-contiguous pool in particular is
often derived at boot from what's actually available then, not a fixed constant).

## Best practices

- Pick the memory pool deliberately for what a resource is actually for — don't default everything
  to general RAM just because it's the biggest pool. GPU resources belong in CDRAM/VRAM if the
  graphics library supports it; DMA-target buffers for hardware decode belong in phycont.
- Track ownership of anything allocated from a small, easily-exhausted pool (phycont especially)
  explicitly in your own code, and don't rely solely on a third-party callback contract to always
  fire cleanup on every exit path.
- Query actual free memory instead of hardcoding totals.
- Be conscious of cached vs uncached when sharing buffers with hardware blocks (GPU, decoders, DMA).
