# vita2d

**vita2d** (`github.com/xerpi/libvita2d`) is a small, high-level 2D graphics library for the Vita,
built directly on top of Sony's native `sceGxm` — not on vitaGL, not on any GL-shaped API at all.
Where [vitaGL](../03-vitagl/README.md) exists to let you run code written against an OpenGL(ES)-
shaped API, vita2d exists for the opposite case: you're not porting anything, you're writing a new
2D app (a UI, a tool, an app-store client, a simple game) directly for the Vita, and you'd rather
call `vita2d_draw_rectangle(...)` than manage vertex buffers, shaders, and matrices yourself.

Real, complete apps ship on it — see [Case study: VHBB](09-case-study-vhbb.md) for a full app-store
client built entirely on vita2d with no other graphics dependency.

## Pages in this section

1. [Overview](01-overview.md) — what vita2d is, where it sits relative to sceGxm/vitaGL, the shape of its API
2. [Initialization & frame lifecycle](02-initialization-and-frame-lifecycle.md) — `vita2d_init`, the draw/swap loop, triple buffering, resource lifetime rules
3. [Primitives & drawing](03-primitives-and-drawing.md) — pixels, lines, rectangles, circles, custom vertex arrays, blend modes
4. [Textures & images](04-textures-and-images.md) — loading PNG/JPEG/BMP, the `draw_texture_*` family, texture atlases
5. [Text & fonts](05-text-and-fonts.md) — three separate text APIs: bundled TTF, and the system's built-in PGF/PVF fonts
6. [Render targets & clipping](06-render-targets-and-clipping.md) — rendering to an off-screen texture, region clipping, escape hatches to raw sceGxm
7. [vita2d vs vitaGL](07-vita2d-vs-vitagl.md) — how to choose between them for a new project
8. [Best practices & pitfalls](08-best-practices-pitfalls.md) — the temp pool, resource-freeing order, color packing, and other easy mistakes
9. [Case study: VHBB](09-case-study-vhbb.md) — a complete, real app-store client built on vita2d, read as a template for "homebrew without porting"

## Scope and sourcing

Function signatures and constants on these pages are read directly from
`libvita2d/include/vita2d.h` and the corresponding `.c` sources in `xerpi/libvita2d` (the
canonical/most-used vita2d implementation, installed via `vdpm`). Behavioral claims not obvious
from the header alone (pool allocation happening inside `draw_texture_*` calls, the exact display
buffer count, etc.) are backed by reading the actual implementation, not inferred — called out
inline where it matters.
