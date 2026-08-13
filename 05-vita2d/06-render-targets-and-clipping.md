# vita2d: Render Targets & Clipping

## Region clipping

```c
void vita2d_set_region_clip(SceGxmRegionClipMode mode, unsigned int x_min, unsigned int y_min, unsigned int x_max, unsigned int y_max);
void vita2d_enable_clipping();
void vita2d_disable_clipping();
int  vita2d_get_clipping_enabled();
void vita2d_set_clip_rectangle(int x_min, int y_min, int x_max, int y_max);
void vita2d_get_clip_rectangle(int *x_min, int *y_min, int *x_max, int *y_max);
```

A rectangular clip that discards drawing outside its bounds — the same concept vitaGL/OpenGL call
"scissor test" and that a UI library like RmlUi's `RenderInterface::EnableScissorRegion` exists
specifically to drive (see the
[RmlUi porting guide](../06-porting-opengl-libraries-to-vitagl/02-case-study-rmlui.md), milestone
M4). If you're building a scrollable panel or a modal dialog on vita2d, this is your `overflow:
hidden` — enable clipping, set the rectangle to the panel's bounds, draw its contents, disable
clipping again (same unbracketed-global-state discipline as `vita2d_set_blend_mode_add`, see
[Primitives & drawing](03-primitives-and-drawing.md)).

## Rendering into a texture instead of the screen

```c
vita2d_texture *vita2d_create_empty_texture_rendertarget(unsigned int w, unsigned int h, SceGxmTextureFormat format);
void vita2d_start_drawing_advanced(vita2d_texture *target, unsigned int flags);
```

Pass a render-target texture to `vita2d_start_drawing_advanced` instead of calling
`vita2d_start_drawing()`, and every draw call until the matching `vita2d_end_drawing()` renders into
that texture instead of the visible framebuffer. Practical uses: pre-rendering a complex, mostly-
static panel once and reusing it as a cheap textured quad on subsequent frames (trading GPU fill
cost for a single texture upload); generating a thumbnail/preview image; simple post-processing by
rendering a scene to a texture and then drawing that texture with a shader (see below) applied.

## Escape hatches to raw sceGxm

```c
SceGxmContext        *vita2d_get_context();
SceGxmShaderPatcher   *vita2d_get_shader_patcher();
const uint16_t        *vita2d_get_linear_indices();
```

vita2d is not a closed box: these return the actual underlying `sceGxm` objects it manages
internally, letting you drop down and issue your own raw `sceGxm` draw calls (custom shaders,
effects vita2d's own API doesn't expose) that compose with vita2d's own drawing in the same frame
— the same "raw draws alongside the high-level library" pattern documented for vitaGL+ImGui in
[Custom rendering integration](../04-imgui/04-custom-rendering-integration.md), just one layer
lower (real `sceGxm` instead of vitaGL). This is an advanced/rarely-needed path — most apps never
touch it — but it's worth knowing it exists before concluding vita2d can't do something and reaching
for a much heavier dependency instead.

## Practical takeaways

- Region clipping is vita2d's scissor test — reach for it for any scrollable/boxed UI content.
- Render-to-texture is a real, supported feature, not something you'd need vitaGL for — useful for
  caching expensive-to-redraw UI as a static texture.
- `vita2d_get_context()`/`get_shader_patcher()` are there precisely so "vita2d doesn't expose X" is
  rarely a hard wall — you can go one level down without abandoning vita2d for everything else.
