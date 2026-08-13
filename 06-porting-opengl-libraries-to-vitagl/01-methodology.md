# General Methodology, Extracted From a Real Port

[imgui-vita](../04-imgui/02-imgui-vita-backend.md) is a genuine, shipped port of Dear ImGui's
rendering backend to vitaGL. Diffing its backend file directly against upstream Dear ImGui's own
`backends/imgui_impl_opengl2.cpp` shows exactly what changes and what doesn't when porting an
OpenGL-based library to vitaGL — and the pattern generalizes well beyond ImGui specifically.

## 1. The library's core is untouched — only the backend file is new

`imgui.cpp`/`imgui.h`/`imgui_draw.cpp` in `Rinnegatamante/imgui-vita` are, algorithmically,
identical to vanilla upstream Dear ImGui — no Vita-specific code anywhere in them. The entire port
is two new files: `imgui_impl_vitagl.cpp/h` (the renderer backend) and `imgui_vita_touch.cpp/h`
(input). This works because Dear ImGui — like most well-architected cross-platform rendering
libraries — was already designed with a renderer-abstraction seam (a backend contract:
`Init`/`NewFrame`/`RenderDrawData`/`Shutdown`) meant to be implemented once per platform.

**Generalized principle**: before writing a single line of Vita-specific code, find the library's
existing backend/renderer-interface seam. If it has one (most serious cross-platform UI/rendering
libraries do — see [RmlUi's `RenderInterface`](02-case-study-rmlui.md#the-three-interfaces-to-implement)
for a second example), the porting task is "implement that interface for vitaGL," not "modify the
library." If it doesn't have one, that's a much bigger, different problem — reconsider whether
porting this specific library is worth it at all.

## 2. Start from the library's *simplest* existing GL backend as a map

Dear ImGui ships both `imgui_impl_opengl2.cpp` (fixed-function/immediate-mode style: matrix stack,
`glEnableClientState`, `glOrtho`) and `imgui_impl_opengl3.cpp` (core-profile: VAOs, shaders,
uniform buffers). imgui-vita's state setup/teardown is nearly line-for-line identical to the GL2
backend, not GL3:

```c
// imgui_impl_opengl2.cpp (upstream)              // imgui_impl_vitagl.cpp (Rinnegatamante)
glEnable(GL_BLEND);                                glEnable(GL_BLEND);
glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);  glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
glEnableClientState(GL_VERTEX_ARRAY);               glEnableClientState(GL_VERTEX_ARRAY);
glMatrixMode(GL_PROJECTION); glPushMatrix();        glMatrixMode(GL_PROJECTION); glPushMatrix();
glOrtho(0, DisplaySize.x, DisplaySize.y, 0, -1, 1); glOrtho(0, DisplaySize.x, DisplaySize.y, 0, 0, 1);
```

This isn't a coincidence — it's a direct, verifiable signal that vitaGL implements enough of the
fixed-function pipeline (matrix stack, client-state arrays, `glOrtho`) to make the *simpler*, older
GL backend the better porting template, even though GL3-core is the "more modern" choice on
desktop. [vitaGL: overview](../03-vitagl/01-overview.md) documents this same subset-not-full-spec
reality from vitaGL's own side.

**Generalized principle**: when a target library offers multiple reference backends of varying
GL-era sophistication, don't default to porting from the newest/most "correct" one. Pick whichever
existing backend's GL feature usage most closely matches what vitaGL actually implements well
(fixed-function-ish, immediate-mode-friendly) — usually the *oldest* still-maintained one.

## 3. Don't force an operation the target API doesn't offer comfortably — reformulate it

Dear ImGui produces indexed geometry (a vertex buffer + a 16-bit index buffer, drawn with a single
`glDrawElements` call on desktop). imgui-vita doesn't try to replicate that call shape on vitaGL —
it resolves every index in a CPU loop and writes out a **flat, deinterleaved (Structure-of-Arrays)
vertex list**, then submits that with vitaGL's own mapped-pointer draw path
(`vglVertexPointerMapped`/`vglDrawObjects`) instead of an indexed draw call. Full walkthrough:
[imgui-vita backend, "Rendering"](../04-imgui/02-imgui-vita-backend.md#rendering-how-a-frames-draw-data-reaches-the-gpu).

**Generalized principle**: when the target API doesn't cleanly support an operation your source
library assumes (here: an arbitrary indexed draw call), the fix is not to fight the target API until
it does — it's to recognize the *reformulation* the target API is actually good at (here: a flat
vertex list via a mapped-pointer draw) and restructure the data to match, even if that means doing
work in your backend code (the CPU-side index resolution) that the source library's own reference
backend didn't have to do.

## 4. Allocate scratch/streaming buffers once, not per frame

imgui-vita allocates its vertex/texcoord/color scratch arrays once at `Init` time
(`VERTEX_BUFFER_SIZE`), and treats them as a manually-wrapped **circular buffer** during rendering
rather than reallocating per frame. This is the same idiom documented independently for
[vita2d's temp pool](../05-vita2d/08-best-practices-pitfalls.md) — a recurring pattern on this
hardware, not specific to any one library.

**Generalized principle**: if the library you're porting has any notion of "compiled/reusable
geometry" in its own API (Dear ImGui doesn't explicitly, but e.g. RmlUi's `CompileGeometry` does —
see the [RmlUi case study](02-case-study-rmlui.md)), lean into it deliberately in your backend
rather than defeating it by reallocating/rebuilding buffers every frame anyway.

## 5. Separate "drawing" from "reading input" into different files

`imgui_impl_vitagl.cpp` (rendering) and `imgui_vita_touch.cpp` (touch/analog-stick-to-mouse
translation) are separate files with no dependency in either direction beyond what `ImGuiIO` needs.
Upstream desktop backends don't need this second file at all — a real mouse/keyboard already
produces events a desktop backend just relays. The Vita has neither, so this translation layer is
new work, but it's kept independent of the rendering work.

**Generalized principle**: build and test the renderer and the input-translation layer separately.
Neither depends on the other being finished, and conflating them makes it much harder to tell
"nothing draws" apart from "input isn't reaching the library" when something goes wrong.

## Summary checklist

- [ ] Find the library's existing render-backend abstraction seam before writing anything.
- [ ] Pick the *simplest* existing reference backend as your porting template, not the most modern one.
- [ ] When an operation doesn't map cleanly to vitaGL, reformulate the data instead of forcing the operation.
- [ ] Allocate scratch/streaming buffers once; reuse or ring-buffer them, never reallocate per frame.
- [ ] Keep the input-translation layer in its own file, independently testable from the renderer.
