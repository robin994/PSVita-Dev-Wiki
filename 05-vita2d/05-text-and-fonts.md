# vita2d: Text & Fonts

vita2d exposes **three separate, parallel text APIs** — same shape (`draw_text`/`draw_textf`/
`draw_text_ls`/`draw_textf_ls`/`text_dimensions`/`text_width`/`text_height`), three different
underlying font sources. Picking between them is a real design decision, not just a naming choice.

## `vita2d_font` — your own TrueType file

```c
vita2d_font *vita2d_load_font_file(const char *filename);
vita2d_font *vita2d_load_font_mem(const void *buffer, unsigned int size);
int vita2d_font_draw_text(vita2d_font *font, int x, int y, unsigned int color, unsigned int size, const char *text);
int vita2d_font_draw_textf(vita2d_font *font, int x, int y, unsigned int color, unsigned int size, const char *text, ...);
```

Renders any `.ttf` you bundle or download, via FreeType internally — the same underlying font tech
covered in the [RmlUi porting guide](../06-porting-opengl-libraries-to-vitagl/02-case-study-rmlui.md)
for a completely different UI library, because FreeType is the standard answer to "rasterize glyphs
from a TrueType font on Vita" regardless of what's drawing the resulting texture. Use this when you
need a specific, consistent typeface across firmware/language settings and are willing to ship
(and mind the license of) a font file.

## `vita2d_pgf` / `vita2d_pvf` — the system's own built-in fonts

```c
vita2d_pgf *vita2d_load_default_pgf();
vita2d_pvf *vita2d_load_default_pvf();
/* + vita2d_load_system_pgf/pvf(numFonts, configs) for language-specific font groups,
   and vita2d_load_custom_pgf/pvf(path) for a specific font file already on the system */
```

These load fonts **already present in the console's firmware** — no file to bundle, and they
automatically cover whatever languages/glyphs the system itself supports (useful for apps that
don't want to hand-manage i18n glyph coverage). PGF and PVF are two different underlying system
font formats/renderers; the API shape is identical between them, so switching from one to the other
is a drop-in change if you find one renders better for your use case.

**Caveat straight from the header comment**: *"PGF functions are weak imports at the moment, they
have to be resolved manually."* Treat this as a real caveat, not boilerplate — it means PGF symbol
resolution isn't guaranteed the same way a normal linked function is, and is worth verifying works
on your actual target firmware/setup rather than assumed.

## Measuring before drawing

All three families expose the same measurement functions:

```c
void vita2d_font_text_dimensions(vita2d_font *font, unsigned int size, const char *text, int *width, int *height);
int  vita2d_font_text_width(vita2d_font *font, unsigned int size, const char *text);
int  vita2d_font_text_height(vita2d_font *font, unsigned int size, const char *text);
```

(and the `_pgf_`/`_pvf_` equivalents, taking a `float scale` instead of an `unsigned int size`) —
needed for anything layout-dependent: centering a title, sizing a button to fit its label,
wrapping text within a panel. There's no built-in text-wrapping helper — measure and break lines
yourself if you need wrapping.

## Practical takeaways

- Three parallel APIs, one shape: `vita2d_font` (your TTF, full typographic control, must ship a
  font file) vs `vita2d_pgf`/`vita2d_pvf` (system fonts, zero bundling, automatic language
  coverage, less control over exact appearance).
- `_ls` variants (`draw_text_ls`) exist specifically for custom line spacing — reach for those over
  manually offsetting `y` per line.
- Always measure with `*_text_dimensions`/`*_text_width`/`*_text_height` before you need pixel-
  accurate layout — don't estimate text size from character count and font size alone.
