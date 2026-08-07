# vitaGL: Initialization & Memory Pools

## vglInit vs vglInitExtended

- **`vglInit(pool_size)`** — the simple form: give it a pool size (bytes) for the general-purpose
  GL memory pool and let it pick reasonable defaults (native 960×544 resolution, no MSAA) for
  everything else.
- **`vglInitExtended(pool_size, width, height, ram_threshold, msaa)`** — the form real applications
  typically use, exposing more control:
  - **`pool_size`** — if `0`, vitaGL computes the general RAM pool size itself from
    `ram_threshold` (see below) instead of you specifying a fixed number.
  - **`width`/`height`** — the framebuffer resolution to actually render at. Most homebrew targets
    the Vita's native 960×544 panel resolution directly rather than rendering at a different
    internal resolution and scaling (see
    [Hardware: GPU architecture](../01-hardware/03-gpu-architecture.md) for why supersampling
    rarely makes sense on this fixed, non-upgradeable hardware).
  - **`ram_threshold`** — when `pool_size` is `0`, this controls how the general RAM pool is sized:
    roughly, `pool = max(0, free_RAM_at_boot - ram_threshold)` — i.e. `ram_threshold` is the amount
    of system RAM *reserved outside* vitaGL's own pool (for the app's own heap, non-GL data
    structures, and so on), not the pool size itself. **Lowering** `ram_threshold` grows vitaGL's
    pool; **raising** it shrinks vitaGL's pool in favor of leaving more general RAM free for
    everything else. This inverse relationship trips people up regularly — read it carefully before
    tuning it, and remember it only affects the **general RAM pool** (`VGL_MEM_RAM`), not any of the
    other pools described below.
  - **`msaa`** — multisampling mode (`SCE_GXM_MULTISAMPLE_NONE` and friends). MSAA costs real GPU
    time and, on tile-based hardware, real tile-buffer bandwidth — see
    [GPU architecture](../01-hardware/03-gpu-architecture.md) for why TBDR-specific performance
    reasoning applies here more than generic "MSAA is cheap on modern GPUs" desktop intuition would
    suggest.

## The memory pools vitaGL exposes

vitaGL doesn't allocate from one undifferentiated heap — it manages (and lets you allocate directly
from) several **distinct pools**, mirroring the hardware's actual physical memory split (see
[Hardware: memory architecture](../01-hardware/04-memory-architecture.md)):

- **`VGL_MEM_RAM`** — general system RAM, sized as described above. Where most textures, vertex
  data, and general GL resources live by default unless you deliberately route something elsewhere.
  This is the pool **every other UI/rendering resource in a typical app competes for** — a detail
  that matters a lot for the next pool below.
  Uncached variant available for CPU/GPU-shared buffers.
- **`VGL_MEM_VRAM`** — the GPU's dedicated CDRAM pool (~128 MB physically separate from system RAM).
- **`VGL_MEM_SLOW`** — the physically-contiguous memory pool, sized from whatever the system reports
  as available at boot (there's no independent "make this bigger" knob the way `VGL_MEM_RAM` has via
  `ram_threshold` — its total size is essentially a fixed, small, boot-time-determined ceiling; see
  [Hardware: memory architecture](../01-hardware/04-memory-architecture.md)). Small and easy to
  exhaust. **Not actually what `SceAvPlayer`'s decoded-frame texture memory needs on real
  hardware**, despite looking like the obvious candidate ("physically contiguous" reads as exactly
  what a hardware decoder should want) — see
  [Hardware: multimedia hardware](../01-hardware/07-multimedia-hardware.md) for what real testing
  against a confirmed-working reference implementation found instead.
- **`VGL_MEM_BUDGET`** — a further pool reserved for coexisting with Sony's native common-dialog UI
  (`sceCommonDialog`) when one is active on screen at the same time as your own rendering.

`vglAlloc(size, pool)`/`vglFree(ptr)` are the direct allocate/free calls against a specific pool;
`vglMemTotal(pool)`/`vglMemFree(pool)` let you query total/free space per pool at runtime — genuinely
useful for diagnosing pool-exhaustion bugs (see
[Common pitfalls](08-common-pitfalls.md) and
[Hardware: multimedia hardware](../01-hardware/07-multimedia-hardware.md) for a concrete real-world
example of a `VGL_MEM_SLOW` exhaustion bug pattern).

## A concrete, real mistake worth knowing about

It's a genuine, observed failure pattern for code to be changed — sometimes during an unrelated
investigation into a *different* bug — from allocating a hardware-DMA-dependent resource (like
AVPlayer's video decode texture memory) from a dedicated pool, to allocating it from `VGL_MEM_RAM`
instead. This compiles fine, and might even *appear* to work in casual testing, but it means that
resource now competes with every ordinary texture and UI resource in the app for the same general
RAM pool, and it likely won't satisfy whatever the consuming hardware block actually needs. Don't
"simplify" a special-purpose allocation onto the general pool just because it compiles and
superficially works.

## `vglInitWithCustomThreshold` and the `cdlg_threshold` gotcha

`vglInitExtended` is actually a thin wrapper around a more general function:
```
vglInitWithCustomThreshold(pool_size, width, height, ram_threshold, cdram_threshold,
                            phycont_threshold, cdlg_threshold, msaa)
```
— reach for this directly when you need to control the `VGL_MEM_VRAM` (CDRAM) sizing, since
`vglInitExtended` hardcodes `cdram_threshold` to `0`, meaning it claims **100% of available CDRAM**
for its own pool at boot. If anything else in your app needs to draw CDRAM directly (outside
vitaGL's own pool - see [Hardware: multimedia hardware](../01-hardware/07-multimedia-hardware.md)
for a real case where this was required), there's nothing left unless you pass a non-zero
`cdram_threshold` yourself.

**The trap**: `cdlg_threshold` (the common-dialog/`sceImeDialog` memory budget) does **not** follow
the same "pool = available − threshold" pattern as the other three. It's computed as
`SCE_KERNEL_MAX_MAIN_CDIALOG_MEM_SIZE − cdlg_threshold` (a fixed constant, not a live free-memory
query) — so `cdlg_threshold = 0` means "reserve the *entire* max budget," the **opposite** of what
`0` means for `ram_threshold`/`cdram_threshold`/`phycont_threshold`. `vglInitExtended` itself passes
`SCE_KERNEL_MAX_MAIN_CDIALOG_MEM_SIZE` (not `0`) here internally, giving a `0`-sized common-dialog
pool by default. Passing `0` for all four thresholds by analogy — reasonable-looking, since it works
correctly for the other three — silently reserves the full common-dialog budget instead, which
broke native `sceImeDialog` text entry outright on real hardware (tapping a text field did nothing,
looked like a frozen app) in a real, shipped regression. `SCE_KERNEL_MAX_MAIN_CDIALOG_MEM_SIZE`
itself isn't in the public headers to reference directly - passing a large sentinel value (e.g.
`0x10000000`) for `cdlg_threshold` reliably reproduces the "reserve nothing" (`0`-sized pool)
default regardless of the constant's actual value. **Treat `cdlg_threshold` as the odd one out and
double check its direction explicitly** rather than assuming it follows the other three thresholds.

## Practical checklist for init

- Pick `width`/`height` matching your actual target resolution (native panel resolution for most
  homebrew — see [Hardware overview](../01-hardware/01-overview.md)).
- Understand the inverse relationship of `ram_threshold` before tuning it — verify with
  `vglMemTotal(VGL_MEM_RAM)` after init that you got the pool size you expected, rather than
  assuming your mental model of the formula is right.
- Route resources to the memory pool their actual consumer requires (GPU-only resources to VRAM
  where it makes sense, DMA-dependent resources to `VGL_MEM_SLOW`) — don't default everything to
  `VGL_MEM_RAM` just because it's the "normal" one.
- Query `vglMemFree`/`vglMemTotal` per pool when diagnosing any "allocation failed" or
  "works once, fails on reuse" bug — see [Common pitfalls](08-common-pitfalls.md).
