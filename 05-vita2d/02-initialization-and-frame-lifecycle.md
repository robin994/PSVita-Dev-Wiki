# vita2d: Initialization & Frame Lifecycle

## Init variants

```c
int vita2d_init();
int vita2d_init_advanced(unsigned int temp_pool_size);
int vita2d_init_advanced_with_msaa(unsigned int temp_pool_size, SceGxmMultisampleMode msaa);
```

Plain `vita2d_init()` sets up the display and a **default 1 MiB temp pool**
(`DEFAULT_TEMP_POOL_SIZE`, read directly from `vita2d.c`) — the scratch allocator every
`vita2d_draw_texture_*`/`vita2d_draw_array*` call draws from, per frame (see
[Best practices & pitfalls](08-best-practices-pitfalls.md) for why this size matters). The
`_advanced` variants exist specifically to raise that budget for apps that draw a lot of dynamic
geometry per frame, or to enable MSAA.

## Display facts (read from source, not the header)

`vita2d.c` hardcodes these for the Vita's fixed display:

| Constant | Value |
|---|---|
| Resolution | 960×544 |
| Color format | `SCE_GXM_COLOR_FORMAT_A8B8G8R8` |
| Display buffer count | **3** (triple buffering) |
| Max pending swaps | 2 |
| Default temp pool | 1 MiB |
| vsync (`vblank_wait`) | on by default |

The Vita's screen resolution is fixed hardware, not a vita2d design choice, but the **triple
buffering** and **default pool size** are vita2d-specific facts worth knowing when reasoning about
frame latency or tuning memory budgets.

## The per-frame sequence

Directly from the upstream sample (`sample/main.c`), the canonical loop is:

```c
vita2d_init();
vita2d_set_clear_color(RGBA8(0x40, 0x40, 0x40, 0xFF));
// ... load textures/fonts here, once ...

while (running) {
    vita2d_start_drawing();
    vita2d_clear_screen();

    // ... vita2d_draw_* calls ...

    vita2d_end_drawing();
    vita2d_swap_buffers();
}

vita2d_fini();               // waits until the GPU has actually finished rendering
vita2d_free_texture(image);  // ...only now is it safe to free GPU-referenced resources
```

`vita2d_start_drawing`/`vita2d_end_drawing` bracket the frame's draw submissions (there's also
`vita2d_start_drawing_advanced(target, flags)` to render into an off-screen texture instead of the
screen — see [Render targets & clipping](06-render-targets-and-clipping.md)).
`vita2d_common_dialog_update()` should be called once per frame too if your app can show any native
Vita dialog (on-screen keyboard, save-data dialogs, etc.) — it pumps that dialog's own rendering.

## The resource-freeing rule this lifecycle implies

The GPU renders asynchronously relative to your CPU-side draw calls — a `vita2d_draw_texture` call
in frame N doesn't mean the GPU has actually *read* that texture's pixels by the time your code
moves on to frame N+1's logic. **`vita2d_fini()` blocks until the GPU has drained its work**, which
is exactly why the sample only frees textures/fonts *after* calling it, not before. The same
concern applies at any point mid-app where you want to free/replace a texture that might still be
in flight: call `vita2d_wait_rendering_done()` first, or you risk the GPU reading freed/reused
memory. This is the same class of GPU/CPU synchronization concern documented for vitaGL in
[vitaGL: rendering pipeline](../03-vitagl/03-rendering-pipeline.md) — different API, same
underlying hardware, same discipline required.

## Practical takeaways

- Default pool size (1 MiB) is a real budget shared by every dynamic draw call in a frame, not just
  your own custom vertex arrays — see [Best practices & pitfalls](08-best-practices-pitfalls.md).
- Triple buffering (3 display buffers, up to 2 pending swaps) is fixed and not configurable per-app.
- Never free a `vita2d_texture`/`vita2d_font` without first being sure the GPU is done with it —
  `vita2d_fini()` guarantees this at shutdown; `vita2d_wait_rendering_done()` gives you the same
  guarantee mid-app.
