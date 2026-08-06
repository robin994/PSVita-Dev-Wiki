# vitaGL: Performance Best Practices

Performance advice for this hardware isn't generic "modern GPU" advice transplanted onto a Vita —
the tile-based deferred rendering architecture underneath sceGxm/vitaGL (see
[Hardware: GPU architecture](../01-hardware/03-gpu-architecture.md)) genuinely changes what's
expensive and what isn't, compared to the immediate-mode GPUs most desktop performance intuition is
built around.

## Batch draw calls; minimize state changes

Because a TBDR GPU processes a full vertex/binning pass across *all* submitted geometry for a frame
before shading anything, and because switching render targets forces a tile-buffer resolve, the
practical advice converges on the same place immediate-mode-GPU advice usually does, but for a
partly different underlying reason: **fewer, larger draw calls with fewer state/texture-binding
changes between them** beats many small draw calls with frequent state churn. If you're rendering a
lot of similar small elements (icons, UI glyphs, particles), batching them into fewer draw calls
(shared texture atlas, single vertex buffer covering multiple elements) is worth the added
bookkeeping complexity on this hardware specifically.

## Minimize render target switches

Switching which framebuffer/render target you're drawing to is comparatively more expensive on a
TBDR than on many immediate-mode GPUs, because it forces the deferred pipeline to flush and resolve
whatever tile state was associated with the previous target. If your rendering approach involves
multiple render-to-texture passes per frame, be deliberate about how many distinct targets you
actually switch between, and in what order — grouping all draws to one target together, rather than
interleaving draws across multiple targets, is meaningfully cheaper here.

## Overdraw is cheaper here than on an immediate renderer, but not free

TBDR architectures generally resolve visibility (hidden-surface removal) before running fragment
shaders in the common case, which makes naive overdraw less costly than it would be on an immediate-
mode GPU forced to shade every submitted fragment regardless of final visibility. This is a real
architectural advantage — but it's not a license to ignore draw ordering and sensible culling
entirely; it just shifts where the actual performance ceiling tends to bite (vertex/binning-pass
cost scaling with *total* submitted geometry, not final visible geometry, tends to matter more here
than fragment-shading overdraw does).

## Use the mapped-pointer draw path as intended

vitaGL's own draw path (`vglVertexPointerMapped`/`vglDrawObjects` — see
[Rendering pipeline](03-rendering-pipeline.md)) is built around directly-GPU-visible memory to avoid
extra copy/upload steps. Structuring your rendering code to reuse and update mapped buffers rather
than reconstructing/reallocating vertex data from scratch every frame is worth doing for anything
performance-sensitive — allocation churn on this hardware's constrained, pool-segmented memory (see
[Memory pools deep dive](06-memory-pools-deep-dive.md)) has real cost beyond just the copy itself.

## Profile whole frames, not individual draw calls

Because of the deferred nature of TBDR rendering, a lot of the actual GPU work for a given draw call
happens later, during tile resolve — not synchronously "at" the point you submitted it. CPU-side
timing wrapped tightly around a single `vglDrawObjects` call is a misleading signal of that call's
real cost. Measure at the level of whole-frame timing (and, if you need finer-grained GPU-side
insight, reach for whatever profiling/capture tooling is available — `SCE_SYSMODULE_RAZOR_CAPTURE`-
based capture tooling exists for this purpose) rather than trusting per-call CPU-side stopwatches.

## MSAA has a real, non-trivial cost

Multisampling (set at `vglInitExtended` time — see
[Initialization & memory pools](02-initialization-memory-pools.md)) costs real tile-buffer bandwidth
on this hardware, more noticeably than "just turn it on, modern GPUs eat this for free" desktop-era
intuition would suggest. Enable it deliberately, having actually measured the tradeoff for your
specific app, rather than defaulting it on because it's a common desktop-GPU default.

## Summary

- Batch draws, minimize per-draw-call state and texture-binding churn.
- Minimize and group render target switches.
- Don't be afraid of overdraw the way you might be on an immediate-mode GPU, but don't rely on it as
  a substitute for sensible geometry submission either.
- Reuse mapped vertex buffers rather than reallocating every frame.
- Profile at the whole-frame level; per-draw-call CPU timing is misleading on a deferred renderer.
- Treat MSAA as a real cost to measure, not a free desktop-era default.
