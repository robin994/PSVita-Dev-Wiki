# vitaGL: Overview

## Why vitaGL exists

Sony's officially documented, native graphics API on the Vita is **sceGxm** — a low-level, explicit
API (build command buffers, manage GPU memory yourself, explicit synchronization) closer in spirit
to a modern explicit graphics API than to classic immediate-mode OpenGL. No official OpenGL(-ES)
driver was ever shipped for the platform. **vitaGL** fills that gap: it's a from-scratch
implementation of an OpenGL-shaped API, translating familiar-looking calls
(`glBindTexture`, `glDrawArrays`, `glEnable`, ...) into the sceGxm command sequences that actually
drive the hardware. This lets code — and, importantly, *other libraries* that expect a GL-shaped API
underneath them, like the [imgui-vita backend](../04-imgui/README.md) — target something familiar
instead of every project having to learn sceGxm from scratch.

## Where it's genuinely OpenGL-like

For a large, commonly-used subset of functionality, vitaGL behaves close enough to real OpenGL
(specifically closer to older, fixed-function-adjacent OpenGL / OpenGL ES 1.x-2.x conventions than
to a modern core-profile GL context) that code written with that mental model mostly works:
texture binding and creation, basic state toggles (`glEnable(GL_BLEND)`, `glBlendFunc`, depth
test/mask), viewport/scissor management, and a recognizable draw-call shape.

## Where it genuinely isn't

- **Non-standard draw entry points.** vitaGL's actual draw path for supplying vertex data is not
  the portable `glVertexPointer`/`glDrawArrays` pair you'd write for desktop GL — it has its own
  **mapped-pointer** functions (`vglVertexPointerMapped`, `vglTexCoordPointerMapped`,
  `vglColorPointerMapped`) paired with its own draw call, **`vglDrawObjects`** — not
  `glDrawArrays`/`glDrawElements` at all. See
  [Rendering pipeline](03-rendering-pipeline.md) for what this actually looks like in practice, and
  note that `vglDrawObjects`'s own signature has genuinely changed across vitaGL's history — see
  [Common pitfalls](08-common-pitfalls.md).
- **A custom multi-pool memory model bolted onto resource creation.** Real OpenGL has no concept of
  "which physical memory pool should this texture's storage come from" — that's entirely a driver
  implementation detail hidden from the API consumer. vitaGL exposes it directly, because on this
  hardware it *matters* which pool a resource lives in (see
  [Memory pools deep dive](06-memory-pools-deep-dive.md)) — this shows up as an explicit
  `VGL_MEM_RAM` / `VGL_MEM_VRAM` / `VGL_MEM_SLOW` / `VGL_MEM_BUDGET` argument threaded through
  various vitaGL-specific allocation-adjacent calls.
- **Init is vitaGL-specific and non-trivial.** There's no windowing-system-provided GL context to
  attach to — `vglInit`/`vglInitExtended` sets up the whole rendering surface, framebuffer, and
  memory pools from scratch as one explicit call. See
  [Initialization & memory pools](02-initialization-memory-pools.md).
- **It's a moving target with real breaking changes.** As an actively-developed community project,
  vitaGL's API surface has changed in backward-incompatible ways between versions (a well-known
  example: `vglDrawObjects`'s argument count). Treat "which exact vitaGL version/commit" as a fact
  worth pinning and documenting, not an interchangeable detail — see
  [Common pitfalls](08-common-pitfalls.md).

## Mental model going forward

Think of vitaGL less as "OpenGL that happens to run on Vita" and more as "a GL-*flavored* API
purpose-built for this hardware, borrowing OpenGL's naming and conventions where convenient and
diverging where the underlying hardware/architecture genuinely calls for it." That framing makes the
non-standard bits (mapped-pointer draws, explicit memory pools) feel like reasonable design
decisions for the platform rather than surprising deviations from a spec you might otherwise expect
strict conformance to.
