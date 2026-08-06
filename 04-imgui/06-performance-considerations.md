# Performance Considerations

Dear ImGui itself is generally lightweight, but running it on genuinely constrained 2011-era mobile
hardware (see [Hardware overview](../01-hardware/01-overview.md)) means a few things worth being
deliberate about that rarely matter on a modern desktop running the same library.

## The vertex scratch buffer is fixed-size

As covered in [The imgui-vita backend](02-imgui-vita-backend.md), the backend's vertex/texcoord/
color scratch arrays are a **fixed-capacity circular buffer** allocated once at init, not a
dynamically-growing structure. A single frame's total UI complexity (total vertices across every
widget, every window, every draw command) that exceeds this capacity causes the write position to
wrap and overwrite earlier data from the *same* frame — a real correctness risk (visual corruption),
not just a performance concern, if you build UI complex enough to hit it. In practice this ceiling is
generous enough for most realistic tool/menu UIs, but it's worth being aware of if you're rendering
something unusually vertex-heavy in a single frame (a very large scrollable table with hundreds of
rows all rendered simultaneously rather than clipped/virtualized, for instance) — and worth knowing
the `_Extended` init variant exists specifically to raise this ceiling if you do hit it.

## Avoid unnecessary per-frame allocation

Because Dear ImGui is immediate-mode, it's easy to fall into patterns that allocate memory every
single frame without noticing — building a `std::string` via concatenation for a widget label,
constructing a temporary `std::vector` to feed a list widget, and so on, all inside the per-frame UI
construction code that runs 30-60+ times a second. On a desktop this rarely registers; on this
hardware's more constrained memory/allocator performance (see
[Hardware: memory architecture](../01-hardware/04-memory-architecture.md)), routine per-frame
allocation churn is a more realistic contributor to frame-time jitter than it would be elsewhere.
Prefer stable, pre-sized buffers/formatting into fixed-size char arrays (`snprintf` into a
stack/member buffer rather than constructing a new `std::string` every frame) for anything
constructed inside the hot per-frame UI path.

## Clip large lists instead of rendering everything

For genuinely large scrollable collections, Dear ImGui's clipper utility (`ImGuiListClipper`) skips
constructing widgets for rows that are scrolled out of view rather than laying out and submitting
draw commands for the entire collection every frame regardless of visibility — worth reaching for
deliberately on this hardware for any list/table that could plausibly grow large (a file browser, an
app catalog), rather than only when a desktop-class profiler flags it as necessary.

## Texture binding changes inside ImGui rendering

Every distinct texture referenced across a frame's ImGui draw commands (including any
`ImGui::Image` calls using your own textures — see
[Custom rendering integration](04-custom-rendering-integration.md)) potentially costs a texture-
bind state change during `RenderDrawData`'s submission — see
[vitaGL: performance best practices](../03-vitagl/07-performance-best-practices.md) for why
minimizing state/texture-binding churn matters more on this hardware's tile-based GPU than generic
desktop-GPU intuition would suggest. If you're rendering many small distinct images inside an ImGui
layout (an icon grid, for instance), the same texture-atlasing advice that applies to raw vitaGL
rendering applies here too.

## Profile whole frames on real hardware

As with vitaGL performance work generally (see
[vitaGL: performance best practices](../03-vitagl/07-performance-best-practices.md)), measure actual
frame time on real hardware rather than assuming Vita3K's performance characteristics represent real-
device performance faithfully — emulator performance and real-hardware performance are not the same
signal, and a UI that feels perfectly responsive in the emulator isn't guaranteed to feel the same on
the actual console.

## Summary

- Know the vertex-buffer ceiling exists and use the `_Extended` init variant if your UI is genuinely
  vertex-heavy.
- Avoid allocating inside per-frame UI construction code — pre-size buffers, format into fixed
  arrays rather than constructing new containers/strings every frame.
- Use `ImGuiListClipper` for large scrollable lists rather than always rendering the full collection.
- Be mindful of texture-binding churn from many distinct `ImGui::Image` calls in one frame.
- Measure real frame time on real hardware, not just emulator performance.
