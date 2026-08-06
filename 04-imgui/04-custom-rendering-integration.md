# Custom Rendering Integration

Very few real applications render *only* ImGui widgets — most mix raw vitaGL draw calls (game/app
content, custom-drawn icons, video frames, background images) with an ImGui overlay or ImGui-driven
UI chrome within the same frame. This page covers how that composition actually works.

## The backend saves and restores state around its own draws

As covered in [The imgui-vita backend](02-imgui-vita-backend.md),
`ImGui_ImplVitaGL_RenderDrawData` saves relevant GL state (texture binding, polygon mode, viewport,
scissor) before submitting its own draw commands and restores it afterward. This means you can, in
principle, freely interleave your own raw vitaGL drawing calls with ImGui's frame lifecycle without
each side corrupting the other's expected state — draw your own background/game content, then call
into Dear ImGui's frame construction and render it on top, and your own subsequent draws (if any)
after that resume from a clean, predictable state rather than whatever ImGui happened to leave bound.

## A typical frame structure mixing both

```
begin_frame()                          // your own per-frame setup
draw_own_background_content()          // raw vitaGL draws — e.g. a video frame texture, game world
ImGui_ImplVitaGL_NewFrame()
ImGui::NewFrame()
    // ... your ImGui::Begin/widgets/End calls ...
ImGui::Render()
ImGui_ImplVitaGL_RenderDrawData(ImGui::GetDrawData())
draw_own_foreground_overlays()         // more raw vitaGL draws, if needed, on top of ImGui
vglSwapBuffers(...)
```

The exact ordering is up to your application's needs — the point is that ImGui's render step is just
*one more thing* your frame does, composable with ordinary vitaGL drawing before and after it,
rather than something that needs to own the entire frame.

## Feeding your own textures into ImGui

`ImGui::Image(texture_id, size)` (and `ImGui::ImageButton`) take a raw texture handle
(`ImTextureID`, effectively a `void*`/GL texture name under the hood) — meaning any texture you've
created through ordinary vitaGL calls (`glGenTextures`/`glTexImage2D`, or even one populated via the
lower-level sceGxm escape hatch described in [vitaGL: textures](../03-vitagl/04-textures.md)) can be
displayed directly inside an ImGui window exactly like any built-in widget. This is how things like
image previews, icon grids, or even a live video-frame texture end up rendered *inside* an ImGui
layout rather than only as separate full-screen background content — genuinely useful for building
tool-style UIs (browsers, galleries, previewers) that mix "real" application-rendered content with
ImGui-driven layout and interaction around it.

## Coordinate systems and scissor/clip rects

Because both your own raw drawing and ImGui's own rendering ultimately go through the same
underlying vitaGL/sceGxm viewport and scissor state, make sure both sides agree on the same
coordinate convention (screen-space origin, Y-axis direction) — a mismatch here typically shows up
as ImGui content rendering upside-down, offset, or with a scissor rect that clips the wrong region
relative to what you'd expect, easy to misdiagnose as an ImGui bug when it's actually a coordinate-
convention mismatch between your own rendering setup and what the backend assumes.

## Practical guidance

- Lean on the backend's state save/restore rather than manually re-establishing GL state before and
  after every ImGui render call yourself — it's already handled.
- Use `ImGui::Image` with your own vitaGL textures to build UI that mixes real rendered content with
  ImGui layout, rather than treating ImGui and "your own graphics" as two entirely separate rendering
  subsystems that never touch.
- Double-check coordinate-convention agreement early if mixed rendering looks subtly wrong (flipped,
  offset, mis-clipped) — it's a common, easy-to-fix root cause for that class of symptom.
