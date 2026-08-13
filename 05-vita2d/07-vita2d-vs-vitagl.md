# vita2d vs vitaGL: Choosing Between Them

Both sit directly on `sceGxm`. Neither is "built on top of" the other. The choice is about what
kind of app you're building and what you're starting from, not about capability in the abstract.

| | **vita2d** | **[vitaGL](../03-vitagl/README.md)** |
|---|---|---|
| API shape | Domain-specific 2D calls (`draw_rectangle`, `draw_texture`) | GLES-like generic graphics API (`glDrawArrays`, shaders, matrices) |
| You manage | Almost nothing — no shaders, no matrices, no VAOs | Shader programs, vertex arrays/buffers, matrix stacks, GL state |
| Best fit | A new 2D app: UI, tool, app-store client, simple game | Porting existing OpenGL(ES) code, or needing real 3D |
| Learning curve if you don't know GL | Gentle — the API already speaks in UI/2D-graphics terms | Steep — you need to actually understand GL concepts first |
| Real-world proof | [VHBB](09-case-study-vhbb.md) — a complete app-store client, no other graphics dependency | [imgui-vita](../04-imgui/02-imgui-vita-backend.md) — Dear ImGui's real, unmodified renderer contract implemented on top of it |

## The actual decision rule

**Are you porting an existing library/codebase that already expects an OpenGL-shaped API** (Dear
ImGui, RmlUi, a game engine)? Then you need vitaGL — the whole point is to satisfy that library's
existing `RenderInterface`/backend contract, and vita2d's API doesn't look anything like OpenGL, so
it can't stand in for it. See the
[porting methodology section](../06-porting-opengl-libraries-to-vitagl/README.md).

**Are you writing new 2D app code, not porting anything?** Start with vita2d. You get a complete,
already-idiomatic-for-2D API with no shader/matrix management, and — per the VHBB case study — it's
proven sufficient for a full, real application including scrollable lists, icons, text, and modal
dialogs. Reach for vitaGL later, specifically, only if you hit something vita2d genuinely can't do
(and check the [render-target/escape-hatch options](06-render-targets-and-clipping.md) first —
vita2d covers more than its function list suggests at first glance).

## Practical takeaway

If you don't already know OpenGL and your goal is "build a Vita app," vita2d is very likely the
right starting point specifically *because* it doesn't require learning OpenGL first — vitaGL earns
its complexity only when you're bridging to code that already assumes GL exists.
