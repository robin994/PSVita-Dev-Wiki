# GPU Architecture

## PowerVR SGX543MP4+

The Vita's GPU is a PowerVR SGX543MP4+ — four SGX cores, part of Imagination Technologies' SGX
family (the same broad lineage found in a lot of 2010-2012-era mobile SoCs). It supports OpenGL
ES 2.0-class features at the hardware level: programmable vertex/fragment shaders, a real
rasterizer pipeline, texture compression formats. Sony did **not** ship a public OpenGL ES driver,
though — the native, documented graphics API is Sony's own **sceGxm**, a much lower-level API
closer to a modern explicit graphics API (Vulkan/Metal-style: you build command buffers, manage
memory yourself, explicitly synchronize) than to classic immediate-mode OpenGL.

This is the whole reason **vitaGL** exists as a project: it's a community-built translation layer
that implements an OpenGL(-ES-like) API surface *on top of* sceGxm, so code written against a
familiar GL-style API can actually run on the hardware. See the [vitaGL section](../03-vitagl/README.md)
for the library itself; this page stays focused on what the underlying hardware is actually doing.

## Tile-Based Deferred Rendering (TBDR)

PowerVR GPUs of this generation are **tile-based deferred renderers**, a fundamentally different
execution model from the immediate-mode rasterizers common on desktop GPUs of the same era:

1. The screen is divided into small tiles (a fixed on-chip tile buffer size, GPU-internal, not
   something application code configures directly).
2. Instead of rasterizing and shading triangles as they're submitted, the GPU first collects *all*
   geometry for a frame (a full "vertex pass"), bins each primitive into which tile(s) it touches.
3. Then, tile by tile, it resolves visibility (hidden-surface removal happens *before* fragment
   shading in the common case — a key power/bandwidth advantage of this architecture) and only then
   runs fragment shaders for the surviving, visible fragments.
4. Finished tiles are written out to the framebuffer in main/video memory.

**Why this matters to you as a programmer**, beyond trivia:
- **Draw call submission timing is deceptive.** On an immediate-mode GPU, a draw call's cost is
  roughly "felt" when submitted. On a TBDR, most of the actual rendering work happens later, during
  tile resolve — so naive CPU-side profiling of "how long did this draw call take" is misleading.
  Profile whole frames, not individual calls.
- **State changes and render target switches are more expensive, relatively, than on immediate
  renderers** — swapping render targets forces the deferred pipeline to flush/resolve the tile
  buffer state associated with the old target. Minimizing render-target switches per frame matters
  more here than "just don't worry about it, modern GPUs are fast" advice from desktop contexts.
- **Overdraw is cheaper than on immediate renderers** (hidden surfaces are often culled before
  shading), but not free, and you shouldn't rely on it as a substitute for sensible draw ordering.
- **Vertex-pass cost scales with *all* geometry submitted for the frame**, even geometry that ends
  up entirely occluded — the binning pass still has to process it.

## sceGxm, briefly

sceGxm exposes: explicit memory-managed vertex/fragment programs (compiled via Sony's Cg-based
shader compiler toolchain — see `vitashark`/`SceShaccCg` in the [VitaSDK section](../02-vitasdk/README.md)),
explicit texture objects (`SceGxmTexture`), explicit command-buffer-style scene begin/end pairs, and
manual synchronization between CPU writes and GPU reads. It is *not* something most homebrew
authors interact with directly day-to-day — vitaGL exists specifically to hide this — but
understanding that sceGxm sits underneath explains several vitaGL quirks (see
[vitaGL: common pitfalls](../03-vitagl/08-common-pitfalls.md)), like non-standard draw-call
functions (`vglDrawObjects`) that don't map 1:1 onto `glDrawArrays`/`glDrawElements`.

## VRAM vs system RAM for graphics resources

The GPU has a dedicated **CDRAM** pool (~128 MB) separate from the ~512 MB of general system RAM.
Textures, render targets, and vertex/index buffers can in principle live in either general RAM or
CDRAM, but CDRAM is the GPU's "close" memory — see
[Memory architecture](04-memory-architecture.md) and
[vitaGL: memory pools deep dive](../03-vitagl/06-memory-pools-deep-dive.md) for the practical
implications of choosing one pool over another for a given resource.

## Display output

Native panel resolution is 960×544 (a somewhat unusual aspect ratio, ~16:9.07). Most 2D UI and a
lot of homebrew game rendering targets this resolution directly rather than rendering at a
different internal resolution and scaling, since there's no benefit to supersampling on a fixed,
non-upgradeable 5" panel and the GPU headroom is limited enough that "render native, skip the
scaling pass" is usually the right call.
