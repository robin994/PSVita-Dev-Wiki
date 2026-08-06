# Dear ImGui / imgui-vita

**Dear ImGui** (`github.com/ocornut/imgui`) is a widely-used, platform-agnostic C++ **immediate-mode
GUI** library — the dominant choice across a huge range of tools, debug overlays, and increasingly
full application UIs, on every major platform. **imgui-vita** is a community Vita port: the
platform+renderer backend (`imgui_impl_vitagl`) and Vita-specific input glue
(`imgui_vita_touch`) needed to run genuine, unmodified Dear ImGui application code on top of vitaGL.

The vast majority of what there is to know about Dear ImGui itself — the immediate-mode paradigm,
the widget API, layout, the ID stack — is **not Vita-specific at all**, and the
[upstream Dear ImGui project's own documentation](https://github.com/ocornut/imgui) is the
authoritative source for that. This section focuses on what's genuinely specific to running it on
this platform: the backend implementation, input handling given the Vita's controller/touch-first
input model (no mouse/keyboard), and UX patterns that make sense for a handheld/controller-driven
device rather than a desktop pointer-driven one.

## Pages in this section

1. [Overview & core concepts](01-overview-core-concepts.md) — the immediate-mode paradigm, frame lifecycle, general Dear ImGui knowledge worth knowing regardless of platform
2. [The imgui-vita backend](02-imgui-vita-backend.md) — `imgui_impl_vitagl`, vertex buffer sizing, how it renders through vitaGL
3. [Input handling](03-input-handling.md) — touch, gamepad-as-navigation, stick-as-mouse, and how they coexist
4. [Custom rendering integration](04-custom-rendering-integration.md) — mixing raw vitaGL draws with ImGui in the same frame
5. [UX patterns for handheld/controller-first apps](05-ux-patterns-handheld.md) — designing for this input model, not a desktop one
6. [Performance considerations](06-performance-considerations.md) — vertex buffer limits, avoiding per-frame allocation churn
7. [Best practices](07-best-practices.md) — a consolidated checklist
