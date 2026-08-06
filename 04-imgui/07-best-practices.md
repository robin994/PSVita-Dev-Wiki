# Dear ImGui / imgui-vita: Best Practices — Consolidated Checklist

## Fundamentals (platform-agnostic, but worth restating)

- Remember immediate mode means the *entire* UI construction genuinely reruns every frame — design
  with that cost model in mind, not a retained-mode mental model.
- Disambiguate same-labeled widgets in the same scope with `##id` suffixes or explicit
  `PushID`/`PopID` — silent ID collisions are one of the most common early Dear ImGui bugs.
- Check a widget's return value (did it change this frame?) inline rather than polling the
  underlying variable separately afterward.

## Vita-specific setup

- Know that only `imgui_impl_vitagl.cpp/h` and `imgui_vita_touch.cpp/h` are genuinely Vita-specific
  — everything else in an imgui-vita checkout is portable, standard Dear ImGui.
- Use the `_Extended` init variant if your UI is vertex-heavy enough to plausibly stress the fixed-
  size scratch vertex buffer; know that buffer wraps rather than grows if exceeded.
- There's no keyboard/text-input backend — plan `sceImeDialog` integration for any text-entry need
  from the start, not as a late addition.

## Input model

- Choose one primary interaction model (touch-first, or gamepad-navigation-first) deliberately
  rather than leaving touch, stick-as-mouse, and gamepad-nav all active by default.
- If going gamepad-nav-first, explicitly disable stick-as-mouse and set
  `ImGuiIO.MouseDrawCursor = false`.
- It's fine — often correct — for your own render loop to read raw `sceCtrl` input directly for
  concerns outside ImGui's own focus-navigation system (top-level screen switching, list scrolling),
  rather than routing everything through ImGui's nav-input system.
- Design layouts (especially grids/lists) with predictable D-pad navigation flow in mind.
- Respect the platform's regional circle/cross confirm-cancel convention rather than hardcoding one.
- Test every modal/dialog flow specifically under gamepad-only input.

## Rendering integration

- Lean on the backend's automatic GL-state save/restore around `RenderDrawData` rather than manually
  re-establishing state yourself.
- Use `ImGui::Image` with your own vitaGL textures to build UI mixing "real" rendered content with
  ImGui layout, rather than treating them as unrelated rendering subsystems.
- Watch for coordinate-convention mismatches (flipped/offset/mis-clipped content) as an early
  diagnostic when custom rendering mixed with ImGui looks subtly wrong.

## Performance

- Avoid allocating inside per-frame UI construction code — format into fixed buffers rather than
  constructing new strings/containers every frame.
- Use `ImGuiListClipper` for large scrollable collections.
- Minimize distinct-texture churn from many `ImGui::Image` calls in one frame, same reasoning as raw
  vitaGL texture-binding costs.
- Measure real frame time on real hardware, not just in Vita3K.

## UX

- Design for handheld/controller-first interaction, not desktop mouse+keyboard assumptions.
- Consider PSTV (no touch at all) when deciding whether any interaction is touch-*only*.
- Consider burn-in mitigation for long-session static UI on OLED hardware (original PCH-1000).
