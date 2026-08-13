# vita2d: Primitives & Drawing

## The basic shapes

```c
void vita2d_draw_pixel(float x, float y, unsigned int color);
void vita2d_draw_line(float x0, float y0, float x1, float y1, unsigned int color);
void vita2d_draw_rectangle(float x, float y, float w, float h, unsigned int color);
void vita2d_draw_fill_circle(float x, float y, float radius, unsigned int color);
```

All colors are a single packed `unsigned int`, built with the `RGBA8` macro:

```c
#define RGBA8(r,g,b,a) ((((a)&0xFF)<<24) | (((b)&0xFF)<<16) | (((g)&0xFF)<<8) | (((r)&0xFF)<<0))
```

Read carefully: the **byte layout in memory is R, G, B, A** (red in the lowest byte), which is why
the internal display format is named `A8B8G8R8` (Sony's naming states the byte order highest-to-
lowest: A, then B, then G, then R) — the same value, described from opposite ends. Always build
colors through the `RGBA8()` macro rather than hand-packing hex constants; getting the shift order
backwards produces a wrong-but-plausible color (channels swapped) rather than an obvious crash,
which makes it an easy silent bug.

## Custom geometry

```c
typedef struct vita2d_color_vertex {
    float x, y, z;
    unsigned int color;
} vita2d_color_vertex;

void vita2d_draw_array(SceGxmPrimitiveType mode, const vita2d_color_vertex *vertices, size_t count);
```

For anything the fixed primitives don't cover (arbitrary polygons, triangle strips/fans, per-vertex
gradients), allocate a `vita2d_color_vertex` array from vita2d's own temp pool and hand it to
`vita2d_draw_array` with a raw `SceGxmPrimitiveType` (`SCE_GXM_PRIMITIVE_TRIANGLES`,
`_TRIANGLE_STRIP`, `_TRIANGLE_FAN`, `_LINES`, ...):

```c
vita2d_color_vertex *v = vita2d_pool_memalign(n * sizeof(*v), sizeof(*v));
// ...fill v[i].x/y/z/color...
vita2d_draw_array(SCE_GXM_PRIMITIVE_TRIANGLE_STRIP, v, n);
```

This is the one place vita2d's API surfaces a raw `sceGxm` type (`SceGxmPrimitiveType`) directly —
everything else is fully wrapped.

## Blend mode: a global toggle, not a stack

```c
void vita2d_set_blend_mode_add(int enable);
```

Switches between normal alpha blending and **additive** blending (useful for glow/highlight
effects). This is a single global flag, not pushed/popped automatically — the upstream sample
explicitly turns it back off after the draw call that needed it:

```c
vita2d_set_blend_mode_add(1);
vita2d_draw_rectangle(40, 60, 200, 60, RGBA8(0, 100, 0, 128));
vita2d_set_blend_mode_add(0);   // <-- easy to forget
```

Forgetting the `(0)` call doesn't error — every subsequent draw call in the frame (and following
frames) silently keeps blending additively until something turns it back off, which reads as "my
UI looks washed out/too bright" rather than an obvious bug.

## Practical takeaways

- Build colors with `RGBA8()`, never by hand — the in-memory byte order (R,G,B,A) is the opposite
  of the format name (`A8B8G8R8`), an easy source of confusion if you try to derive it from the
  name alone.
- `vita2d_draw_array` is the escape hatch for any shape the fixed primitives don't cover — same
  pool-allocation discipline as textured draws (see
  [Best practices & pitfalls](08-best-practices-pitfalls.md)).
- Treat `vita2d_set_blend_mode_add` like any other unbracketed global GL-style state toggle: set it,
  use it, turn it back off immediately after — don't leave it set across draw calls that don't need it.
