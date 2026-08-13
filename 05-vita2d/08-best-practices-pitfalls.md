# vita2d: Best Practices & Pitfalls

## The temp pool is a shared, per-frame budget — not just for your own geometry

`vita2d_pool_malloc`/`_memalign` back a fixed-size scratch arena (1 MiB by default, see
[Initialization & frame lifecycle](02-initialization-and-frame-lifecycle.md)), reset each frame via
`vita2d_pool_reset()` (called internally — you don't normally call it yourself). The easy mistake is
assuming this budget is only consumed by geometry *you* explicitly allocate with
`vita2d_pool_memalign`. It isn't: **every single `vita2d_draw_texture*` call also allocates its
quad's vertices from this same pool internally** (confirmed in `vita2d_texture.c`). A screen with
many textured draws (an icon grid, a long list with thumbnails) contributes to the same budget as
any custom `vita2d_draw_array` geometry you submit yourself. If you exhaust the pool mid-frame,
raise it via `vita2d_init_advanced(temp_pool_size)` at startup rather than trying to hand-optimize
draw call count first.

This is architecturally the same pattern as [imgui-vita's own fixed-size, circular scratch
buffer](../04-imgui/02-imgui-vita-backend.md) — a recurring idiom on this hardware: allocate a
scratch region once, reuse/reset it every frame, rather than `malloc`/`free` per draw call.

## Resource lifetime: don't free what the GPU might still be reading

Covered in depth in
[Initialization & frame lifecycle](02-initialization-and-frame-lifecycle.md#the-resource-freeing-rule-this-lifecycle-implies):
`vita2d_free_texture`/`vita2d_free_font` are safe once you know the GPU is done referencing that
resource (guaranteed at shutdown after `vita2d_fini()`, or mid-app after
`vita2d_wait_rendering_done()`), not simply once your own CPU-side draw calls for that frame have
returned.

## Color packing

Always go through the `RGBA8()` macro (see
[Primitives & drawing](03-primitives-and-drawing.md#the-basic-shapes)) — the in-memory byte order
(R,G,B,A) is the reverse of the display format's name (`A8B8G8R8`), a mismatch that's easy to get
backwards if you ever hand-derive a packed color value from the format name instead of the macro.

## `_part` texture coordinates are pixel-space

`vita2d_draw_texture_part`'s `tex_x/tex_y/tex_w/tex_h` are source-texture **pixel** coordinates
(vita2d divides by texture width/height internally to get UVs), not normalized 0–1 UV — see
[Textures & images](04-textures-and-images.md). Passing normalized coordinates here silently
samples the wrong (usually tiny, top-left) region of the texture rather than erroring.

## Global toggles don't reset themselves

`vita2d_set_blend_mode_add(1)`, `vita2d_enable_clipping()`, and similar calls set state that
persists across draw calls and frames until explicitly turned back off — none of vita2d's drawing
functions save/restore this kind of state around themselves (unlike, say,
[imgui-vita's `RenderDrawData`](../04-imgui/02-imgui-vita-backend.md), which does save/restore GL
state around its own submission because it has to compose with arbitrary surrounding app code).
Bracket every non-default mode toggle with its own reset immediately after the draw calls that need
it.

## `sce_sys` PNG assets are stricter than vita2d's own PNG loader

Separate concern from `vita2d_load_PNG_file` (which uses `libpng` directly and isn't picky about
palette format): PNGs consumed by the **system firmware itself** — `sce_sys/icon0.png`,
`sce_sys/livearea/contents/*.png` — are parsed by Sony's own installer/LiveArea code, which is
documented (by real-world report, in VHBB's own README) to require an **indexed palette**, and to
be able to crash VPK installation outright if a PNG exported by a typical image editor doesn't meet
that expectation. This is a packaging-time constraint on your app's icon/LiveArea assets, not on
textures you load and draw at runtime with vita2d — don't conflate the two.

## Practical takeaways

- Size your temp pool for total per-frame draw-call volume, not just your own custom geometry.
- Never free a texture/font without being sure the GPU has finished with it.
- Build colors only via `RGBA8()`; pass pixel (not UV) coordinates to `_part` texture calls.
- Every global toggle (blend mode, clipping) needs a matching reset — vita2d won't do it for you.
- Run `sce_sys` PNGs through a palette-aware optimizer (pngquant, or the bundled PNGO tool VHBB
  ships under `/tools/`) before packaging, independently of how you handle runtime textures.
