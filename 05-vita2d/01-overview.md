# vita2d: Overview

## What it actually is

vita2d is a C library exposing roughly 90 functions, all prefixed `vita2d_`, covering: display
init/lifecycle, primitive drawing (pixels/lines/rectangles/circles/custom vertex arrays), texture
loading and drawing (PNG/JPEG/BMP), and text rendering (bundled TrueType fonts, plus the console's
own built-in system fonts). It is implemented directly on `sceGxm` (Sony's native low-level GPU
API) — internally it compiles its own tiny set of `sceGxm` shaders (a plain-color program and a
textured program) and manages its own vertex/index buffers, but none of that is exposed to
application code. You never see a `SceGxmContext`, a shader, or a matrix unless you deliberately
reach for one of the escape hatches (see
[Render targets & clipping](06-render-targets-and-clipping.md)).

This is a different design point from [vitaGL](../03-vitagl/01-overview.md), which re-implements a
*subset of an existing, generic API* (OpenGL/GLES) on top of `sceGxm` so that GL-shaped code can run
mostly unmodified. vita2d isn't trying to be API-compatible with anything — its function names are
domain-specific (`vita2d_draw_rectangle`, not `glBegin(GL_QUADS)`), so there's no existing codebase
"ported" to it; you write directly against it.

## Where it sits in a typical app

```
Your app code
      |
   vita2d           <-- what this section documents
      |
   sceGxm           <-- Sony's native GPU API (not documented in this wiki section by section;
      |                  see the vitaGL section's own sceGxm background where relevant)
  GPU (SGX543MP4+)
```

There is no vitaGL in this picture at all for a vita2d-only app — the two libraries are
alternatives, not layers on top of each other. See [vita2d vs vitaGL](07-vita2d-vs-vitagl.md) for
when to pick which.

## The shape of the public API

Everything is declared in a single header, `vita2d.h`. The function families, in the order they
matter for a first read:

- **Lifecycle**: `vita2d_init*`, `vita2d_fini`, `vita2d_start_drawing`/`vita2d_end_drawing`,
  `vita2d_swap_buffers` — see [Initialization & frame lifecycle](02-initialization-and-frame-lifecycle.md).
- **Primitives**: `vita2d_draw_pixel`/`_line`/`_rectangle`/`_fill_circle`/`_array` — see
  [Primitives & drawing](03-primitives-and-drawing.md).
- **Textures**: `vita2d_texture` (opaque struct), `vita2d_load_PNG/JPEG/BMP_*`, the
  `vita2d_draw_texture_*` family — see [Textures & images](04-textures-and-images.md).
- **Text**: `vita2d_font` (your own TTF), `vita2d_pgf`/`vita2d_pvf` (the system's own built-in
  fonts) — see [Text & fonts](05-text-and-fonts.md).
- **Pool allocator**: `vita2d_pool_malloc`/`_memalign`/`_free_space`/`_reset` — a per-frame scratch
  allocator used both internally by vita2d and available to application code for custom geometry;
  covered throughout, with the full picture in
  [Best practices & pitfalls](08-best-practices-pitfalls.md).

## Practical takeaway

If your mental model going in is "a Canvas-like 2D drawing API with a texture/font layer bolted
on," you already understand vita2d's shape. There is no scene graph, no retained-mode drawing list
— every `vita2d_draw_*` call submits geometry for the *current* frame immediately, the same
immediate-mode discipline as raw GL calls between `glBegin`/`glEnd`, just with a much smaller,
purpose-built vocabulary.
