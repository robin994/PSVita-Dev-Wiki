# vitaGL

**vitaGL** (`github.com/Rinnegatamante/vitaGL`, and various forks/variants in the wild) is a
community-built graphics library that implements an OpenGL(-ES-like) API surface on top of Sony's
native, much lower-level **sceGxm** API (see
[Hardware: GPU architecture](../01-hardware/03-gpu-architecture.md) for what sceGxm actually is and
why the Vita has no official OpenGL driver of its own). It's the graphics library most Vita homebrew
that wants portable-feeling OpenGL code reaches for, rather than writing directly against sceGxm.

It is **not** a complete, spec-conformant OpenGL implementation — it's a pragmatic subset covering
what homebrew commonly needs, with some genuinely non-standard extensions of its own (mapped-pointer
draw calls, a custom multi-pool memory-type argument bolted onto texture/buffer creation) that don't
exist in real OpenGL at all. Code written against it isn't blindly portable to a desktop GL context
without adjustment, and vice versa.

## Pages in this section

1. [Overview](01-overview.md) — what vitaGL is, how it relates to real OpenGL, its non-standard bits
2. [Initialization & memory pools](02-initialization-memory-pools.md) — `vglInit`/`vglInitExtended`, pool sizing
3. [Rendering pipeline](03-rendering-pipeline.md) — the mapped-pointer draw path, `vglDrawObjects`, state management
4. [Textures](04-textures.md) — creation, formats, filtering, sceGxm interop
5. [Shaders](05-shaders.md) — Cg-based custom shaders vs the fixed-function-style default path
6. [Memory pools deep dive](06-memory-pools-deep-dive.md) — RAM vs VRAM vs physically-contiguous vs budget pools, choosing correctly
7. [Performance best practices](07-performance-best-practices.md) — batching, state-change costs, TBDR-aware advice
8. [Common pitfalls](08-common-pitfalls.md) — version drift, non-standard API surface, real gotchas from the field
