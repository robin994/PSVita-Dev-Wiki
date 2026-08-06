# Dear ImGui: Overview & Core Concepts

This page is general Dear ImGui knowledge, applicable on any platform it runs on — worth knowing
solidly before the Vita-specific pages that follow.

## Immediate mode, in contrast to retained mode

Most traditional GUI toolkits (Qt, GTK, Win32, most game-engine UI systems) are **retained-mode**:
you construct persistent widget objects once, hold onto them, and mutate their properties over time;
the toolkit owns their state and redraws them as needed. **Dear ImGui is immediate-mode**: there are
no persistent widget objects at all. Every frame, your own code calls functions like
`ImGui::Button("Save")` or `ImGui::SliderFloat("Volume", &volume, 0.0f, 1.0f)` directly inline with
your application logic — the call *is* the widget, for exactly that one frame, and it directly reads
from and writes to *your own* application variables (a `float&`, a `bool&`) rather than the toolkit
owning a separate parallel copy of that state.

**What this buys you**: your UI code and your application logic are the same code, interleaved
naturally — `if (ImGui::Button("Delete")) { delete_selected_item(); }` reads exactly like it behaves,
with no separate signal/slot wiring, no callback registration boilerplate, no synchronization between
"the widget's value" and "the actual application state" (there's only ever one copy of the value,
which the widget reads from and writes to directly). This is a large part of why it's so popular for
fast-iterating tool UIs and debug overlays — there's very little ceremony between "I want a slider
for this value" and having one.

**What it costs you**: because nothing is retained, *you* re-run the full UI-construction code every
single frame, and any layout/sizing that depends on "how big was this element last frame" (which
does come up — text wrapping, auto-sizing containers) is resolved via a one-frame-lag internal
mechanism rather than an up-front layout pass the way a retained-mode toolkit might do it. This is
rarely a problem in practice, but understanding that "immediate mode" really does mean "the entire
UI is rebuilt from scratch, every frame" (not just "redrawn," but the actual widget construction
calls all re-run) is the single most important mental model shift for anyone new to it.

## The frame lifecycle

Every frame follows the same three-step shape:

1. **`NewFrame()`** — tells Dear ImGui a new frame is starting; internally resets/prepares the
   frame's input state and draw-command accumulation.
2. **Your UI construction code** — the actual `ImGui::Begin("Window")` / widget calls /
   `ImGui::End()` blocks, run inline with the rest of your per-frame application logic, in whatever
   order and however many times you need across the frame.
3. **`Render()`** followed by **actually drawing** the result — `Render()` finalizes the frame's
   accumulated draw data into an `ImDrawData` structure (vertex/index buffers, per-draw-command
   texture/clip-rect info); a separate renderer-backend-specific step then actually submits that
   draw data to the GPU (this is exactly the part a platform backend like imgui-vita's
   `imgui_impl_vitagl` implements — see [The imgui-vita backend](02-imgui-vita-backend.md)).

Getting this ordering right matters: widget calls made *before* `NewFrame()` or *after* `Render()`
in a given frame are a common source of confusing bugs for newcomers.

## Windows, widgets, and layout

- **`Begin("Title")` / `End()`** delimits a window; most widget calls between them are laid out
  automatically top-to-bottom (a simple, implicit vertical-stack layout by default), with explicit
  layout helpers (`SameLine()`, columns, tables) available when you need something other than a
  plain vertical stack.
- Widgets take their **current value by reference** (`ImGui::Checkbox("Enabled", &flag)`) and return
  a bool indicating whether the value *changed this frame* — the idiomatic pattern for reacting to
  user interaction is checking that return value inline (`if (ImGui::Checkbox(...)) { on_change(); }`
  ), not polling the underlying variable separately afterward.

## The ID stack (and why it exists)

Because nothing is retained between frames, Dear ImGui needs *some* way to associate a specific
widget instance with persistent state that genuinely does need to survive across frames internally
(is this text field currently focused, is this tree node expanded, is this combo box open) — this is
solved with an internal **ID system**, derived by default from the widget's label text combined with
its position in the current ID-stack scope (window names, `PushID`/`PopID` scopes, and loop-index
disambiguation when generating widgets in a loop all feed into this). The classic gotcha: **two
widgets with the same visible label in the same ID scope collide** — calling `ImGui::Button("Delete")`
twice in the same window without disambiguation makes Dear ImGui treat them as the *same* interactive
element internally, causing confusing state bugs. The standard fix is the `##` suffix convention
(`ImGui::Button("Delete##item1")`, `ImGui::Button("Delete##item2")` — text after `##` is used for ID
computation but not displayed) or explicit `PushID(index)`/`PopID()` scoping when generating widgets
in a loop over a collection.

## Where to go deeper

This page covers the concepts that come up constantly and are easy to get subtly wrong on first
contact. For the full widget catalog, styling system, and advanced features (docking, viewports,
tables), the [upstream Dear ImGui repository](https://github.com/ocornut/imgui) and its bundled
`imgui_demo.cpp` (a genuinely excellent, comprehensive interactive reference of nearly every widget
and pattern the library offers) are the best next stop — none of that is platform-specific, so it
applies exactly as-is on Vita once the [backend](02-imgui-vita-backend.md) is wired up.
