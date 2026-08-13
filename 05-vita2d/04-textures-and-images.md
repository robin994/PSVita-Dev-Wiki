# vita2d: Textures & Images

## Loading

```c
vita2d_texture *vita2d_load_PNG_file(const char *filename);
vita2d_texture *vita2d_load_PNG_buffer(const void *buffer);
vita2d_texture *vita2d_load_JPEG_file(const char *filename);
vita2d_texture *vita2d_load_JPEG_buffer(const void *buffer, unsigned long buffer_size);
vita2d_texture *vita2d_load_BMP_file(const char *filename);
vita2d_texture *vita2d_load_BMP_buffer(const void *buffer);
```

Decoding is done with bundled `libpng`/`libjpeg` (the same libraries VitaSDK ships prebuilt for
Vita, and the same ones a hand-rolled renderer like the one in
[imgui-vita's backend](../04-imgui/02-imgui-vita-backend.md) would need to reach for itself —
vita2d gives you this for free). The `_buffer` variants exist so you can decode an image embedded
directly into your binary (see the `_binary_<name>_start` linker-symbol pattern used for bundled
assets, also used by VHBB — [case study](09-case-study-vhbb.md)) instead of requiring a real file
on the memory card.

`vita2d_texture` itself is an opaque-ish struct wrapping a `SceGxmTexture`, its backing memory
`SceUID`, and (for render-target textures) a `SceGxmRenderTarget`/color+depth surfaces — you
generally don't touch its fields directly, but `vita2d_texture_get_width/height/stride/format/
datap/palette` exist for the cases you need to inspect it.

## Creating empty/render-target textures

```c
vita2d_texture *vita2d_create_empty_texture(unsigned int w, unsigned int h);
vita2d_texture *vita2d_create_empty_texture_format(unsigned int w, unsigned int h, SceGxmTextureFormat format);
vita2d_texture *vita2d_create_empty_texture_rendertarget(unsigned int w, unsigned int h, SceGxmTextureFormat format);
```

The `_rendertarget` variant is how you get an off-screen render target — see
[Render targets & clipping](06-render-targets-and-clipping.md).

## The `draw_texture_*` family: three independent axes, not fifteen concepts

The header declares roughly fifteen `vita2d_draw_texture*` functions. They look intimidating listed
flat, but they're really **three independent parameters composed together**:

1. **Transform**: plain / `_rotate` / `_rotate_hotspot` (rotate around an arbitrary pivot, not just
   the texture's center) / `_scale` / `_scale_rotate` / `_scale_rotate_hotspot`.
2. **Region**: whole texture, or `_part` (draw only a sub-rectangle — the basis for sprite sheets /
   texture atlases).
3. **Tint**: plain, or `_tint_*` (multiply the texture's color by a given `RGBA8` color — e.g. for a
   hover/selection highlight, a fade-in, or a flash effect).

```c
void vita2d_draw_texture(const vita2d_texture *texture, float x, float y);
void vita2d_draw_texture_part(const vita2d_texture *texture, float x, float y,
                               float tex_x, float tex_y, float tex_w, float tex_h);
void vita2d_draw_texture_tint_part_scale_rotate(const vita2d_texture *texture, float x, float y,
                               float tex_x, float tex_y, float tex_w, float tex_h,
                               float x_scale, float y_scale, float rad, unsigned int color);
```

**`tex_x`/`tex_y`/`tex_w`/`tex_h` are in source-texture pixel coordinates, not normalized 0–1 UV** —
confirmed directly in `vita2d_texture.c`, where the internal implementation divides by the
texture's own width/height to compute UVs (`u0 = tex_x / texture_width`, etc.). You pass pixel
rectangles exactly like you'd index into a sprite sheet image, and vita2d does the UV math for you.

For anything the fixed variants don't cover (arbitrary quads with per-vertex UVs, a wavy/warped
texture effect), `vita2d_draw_array_textured` takes a `vita2d_texture_vertex` array (`x,y,z,u,v`)
the same way `vita2d_draw_array` does for plain color geometry.

## Practical takeaways

- Don't be intimidated by the size of the `draw_texture_*` family — it's transform × region × tint,
  memorize the three axes instead of fifteen function names.
- `_part` coordinates are pixel-space, matching how you'd think about a sprite sheet, not UV-space.
- Every `vita2d_draw_texture*` call allocates its vertex data from vita2d's shared temp pool
  (confirmed in `vita2d_texture.c`) — a screen with many textured draws contributes to the same pool
  budget as any custom geometry you submit yourself. See
  [Best practices & pitfalls](08-best-practices-pitfalls.md).
