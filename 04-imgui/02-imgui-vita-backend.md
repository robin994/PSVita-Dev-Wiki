# The imgui-vita Backend

## What's actually Vita-specific in imgui-vita

The bulk of an imgui-vita checkout is **vanilla upstream Dear ImGui** — `imgui.cpp`/`imgui.h`/
`imgui_draw.cpp`/`imgui_internal.h` plus its bundled `stb_rect_pack`/`stb_textedit`/`stb_truetype`
dependencies (font rasterization) — unmodified library code, not documented further here (see
[Overview & core concepts](01-overview-core-concepts.md) and upstream's own docs for that). The
genuinely Vita-specific, backend-only part is small and self-contained: **`imgui_impl_vitagl.cpp/h`**
(the combined platform+renderer backend) and **`imgui_vita_touch.cpp/h`** (touchscreen input
handling). `imgui_vita.h` is a two-line umbrella include (`imgui.h` + `imgui_impl_vitagl.h`) — the
one header application code actually includes.

## Init

**`ImGui_ImplVitaGL_Init()`** (or the **`_Init_Extended()`** variant most real applications use)
allocates scratch vertex/texcoord/color arrays via plain `malloc()` — **not** through vitaGL's own
GPU memory pools; this is ordinary newlib heap memory — sized to a fixed capacity
(`VERTEX_BUFFER_SIZE`, larger for the `_Extended` variant). This is a **fixed-size circular buffer**:
every draw command's vertices get appended as the frame is rendered, and once the running write
position exceeds the buffer's capacity minus a safety margin, it **wraps back to the start** rather
than growing dynamically. A UI complex enough to exceed this within a single frame would start
overwriting earlier vertex data from the *same* frame's draw list — see
[Performance considerations](06-performance-considerations.md) for what this means practically and
when it's actually a real risk versus a non-issue.

## Rendering: how a frame's draw data reaches the GPU

**`ImGui_ImplVitaGL_RenderDrawData(ImDrawData*)`** walks Dear ImGui's finalized per-frame draw
command lists and manually unpacks each indexed vertex into the flat scratch buffers described above
(position/texcoord/color deinterleaved into separate arrays — Dear ImGui's own internal vertex
format is interleaved/indexed, so this unpacking step is real, non-trivial work the backend does
every frame), then submits the result via vitaGL's own non-standard mapped-pointer draw path
(`vglVertexPointerMapped`/`vglTexCoordPointerMapped`/`vglColorPointerMapped` +
`vglDrawObjects(GL_TRIANGLES, ...)`) — **not** the portable `glDrawArrays`/`glVertexPointer` path
that most other Dear ImGui backends (targeting real OpenGL) use, because vitaGL doesn't really offer
that as the primary efficient path on this hardware. See
[vitaGL: rendering pipeline](../03-vitagl/03-rendering-pipeline.md) for why that draw path looks the
way it does.

The backend saves and restores GL texture binding, polygon mode, viewport, and scissor state around
its own draw submission, so it composes cleanly with whatever your own application's raw vitaGL
draws were doing immediately before/after in the same frame — see
[Custom rendering integration](04-custom-rendering-integration.md) for how to lean on that
deliberately.

## Input: three independent, toggleable sources

**`ImGui_ImplVitaGL_NewFrame()`** (called once per frame, alongside Dear ImGui's own `NewFrame()`)
feeds `ImGuiIO` from up to three independently-toggleable input sources — see
[Input handling](03-input-handling.md) for the full picture of how they interact and how to choose
which to enable for a given app. There is **no keyboard/text-input backend at all** — this platform
has no physical keyboard, and the backend doesn't attempt to synthesize one from touch; text entry
needs a different mechanism entirely (Sony's native `sceImeDialog` — see
[VitaSDK: system libraries](../02-vitasdk/05-system-libraries.md)), invoked by your own application
code outside of ImGui's own widget system, not something ImGui itself drives on this platform.

## Practical takeaways

- Treat `imgui_impl_vitagl.cpp`/`imgui_vita_touch.cpp` as the entire Vita-specific surface area —
  everything else in an imgui-vita checkout is portable, standard Dear ImGui you can reason about
  using upstream documentation.
- Know that the vertex scratch buffer is fixed-size and circular, not dynamically growing — relevant
  if you're building UI complex enough to plausibly stress it.
- There's no built-in text-input widget backend; plan for `sceImeDialog` integration explicitly for
  any text-entry need.
