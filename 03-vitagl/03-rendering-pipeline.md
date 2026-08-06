# vitaGL: Rendering Pipeline

## The mapped-pointer draw path

Real (desktop/ES) OpenGL's classic immediate-mode-adjacent draw path is roughly:
`glVertexPointer`/`glTexCoordPointer`/`glColorPointer` (pointing at client-side or bound-buffer
vertex data) followed by `glDrawArrays`/`glDrawElements`. vitaGL implements its own, differently-
named equivalents instead:

- **`vglVertexPointerMapped(ptr)`**
- **`vglTexCoordPointerMapped(ptr)`**
- **`vglColorPointerMapped(ptr)`**
- **`vglDrawObjects(mode, count, ...)`** — the actual draw call.

These aren't drop-in aliases for the standard GL functions — they're vitaGL's own API, tuned for how
it manages GPU-visible memory under the hood ("mapped" here referring to memory that's directly
GPU-accessible without an extra copy/upload step, which matters a lot given how constrained and
pool-segmented Vita memory is — see
[Memory pools deep dive](06-memory-pools-deep-dive.md)). Any code — including other libraries built
on top of vitaGL, like the [imgui-vita rendering backend](../04-imgui/README.md) — that wants to
submit geometry efficiently on this platform goes through this path rather than the portable
`glDrawArrays` entry point.

## vglDrawObjects's signature has genuinely changed across versions

This is worth calling out prominently because it's a real, documented source of "compiles, doesn't
render" bugs: `vglDrawObjects` has existed with **both a 2-argument form** (`mode, count`) **and a
3-argument form** (`mode, count, implicit_wvp` — an extra flag controlling whether vitaGL applies an
implicit world-view-projection matrix transform) at different points in the project's history.
Because both forms are valid C function calls with different arities, a version mismatch here
doesn't reliably fail to compile — it's exactly the kind of thing where you might drop the third
argument to satisfy a newer header and get a binary that builds cleanly but doesn't actually render
anything on screen (or renders incorrectly), because the semantics of the call changed underneath
you, not just its arity. See [Common pitfalls](08-common-pitfalls.md) for how to spot and avoid
this.

## State management

For the subset of standard GL state vitaGL implements, it behaves the way you'd expect from
fixed-function-era OpenGL: `glEnable`/`glDisable` toggles (`GL_BLEND`, `GL_DEPTH_TEST`,
`GL_ALPHA_TEST`, `GL_SCISSOR_TEST`, ...), `glBlendFunc`, `glDepthMask`, `glPolygonMode`,
`glViewport`/`glScissor`. A library or application layering its own rendering on top of vitaGL
(again, imgui-vita is the canonical example) typically **saves and restores** the relevant state
around its own draw calls, so it composes cleanly with whatever the surrounding application code was
doing immediately before/after — a good pattern to follow in your own rendering code too if you're
mixing multiple rendering subsystems (a custom 2D UI layer plus an ImGui overlay, for instance)
within the same frame.

## Framebuffer / swap chain

`vglSwapBuffers(has_commondialog)` is the frame-end call — presents the completed frame and prepares
the next one. Its boolean argument controls interop with Sony's native common-dialog overlay system
(`sceCommonDialog` — see [VitaSDK: system libraries](../02-vitasdk/05-system-libraries.md)), **not**
vsync — a naming/semantic trap worth knowing about explicitly, since "boolean argument to a
buffer-swap call" reads as a very plausible vsync toggle to anyone pattern-matching from other
graphics APIs, and getting this wrong (treating it as vsync when it's actually "is a native dialog
currently up") is an easy, non-obvious mistake to make and then not notice, since the visible
symptom (if any) is subtle rather than an obvious crash.

## GL vs sceGxm underneath

Every one of these calls is, under the hood, building and submitting sceGxm command-buffer state —
you generally don't need to think about that layer directly (that's the whole point of using
vitaGL), but understanding it explains *why* certain operations that feel "free" on a desktop GPU
(switching render targets, certain state changes) have a real cost here — see
[Hardware: GPU architecture](../01-hardware/03-gpu-architecture.md) for the tile-based-deferred-
rendering angle, and [Performance best practices](07-performance-best-practices.md) for what that
means concretely for how you should structure draw calls.
