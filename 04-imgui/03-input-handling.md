# Input Handling

The Vita has no mouse and no keyboard — every input model Dear ImGui was originally designed around
(desktop mouse pointer + keyboard) has to be substituted with something else entirely. imgui-vita's
backend offers **three independent, individually-toggleable** input sources feeding `ImGuiIO` each
frame, and understanding how they interact (and which combination makes sense for your app) is the
core Vita-specific input topic.

## The three sources

- **Touch** (`ImGui_ImplVitaGL_TouchUsage(bool)`) — front (and optionally rear) panel touch, mapped
  to a virtual mouse position via `imgui_vita_touch.cpp`. Direct, tap-to-position interaction — the
  closest analogue to how touch UIs generally work elsewhere.
- **Gamepad-as-navigation** (`ImGui_ImplVitaGL_GamepadUsage(bool)`) — maps Vita face buttons/D-pad
  directly onto Dear ImGui's built-in **gamepad navigation mode**
  (`ImGuiNavInput_*` slots: cross/circle typically to Activate/Cancel, D-pad to directional
  movement, shoulder buttons to focus-prev/next). This drives Dear ImGui's own focus-based
  navigation system — moving a highlighted focus between widgets and activating the focused one —
  rather than emulating a pointer at all.
- **Left-stick-as-mouse** (`ImGui_ImplVitaGL_MouseStickUsage(bool)`, on by default in the backend
  unless explicitly disabled) — the left analog stick drives a virtual mouse cursor position,
  shoulder triggers act as click — a pointer-emulation approach, distinct from the navigation-focus
  approach above.

## They don't automatically exclude each other — you choose

All three can, in principle, be active simultaneously, but most real applications deliberately pick
a subset matching their actual intended control scheme rather than leaving everything on by default:
a controller-first app (menus, settings, file browsers navigated by D-pad/buttons) typically wants
**gamepad navigation on**, **stick-as-mouse and touch off** (also setting
`ImGuiIO.MouseDrawCursor = false`, since there's no cursor metaphor to draw if you're not using
pointer-style interaction at all) — favoring Dear ImGui's built-in focus-navigation model as the
single, consistent way the UI responds to input, rather than mixing a cursor metaphor in alongside
it.

## Your own input code doesn't have to (and often shouldn't) route entirely through ImGui

It's a common and reasonable pattern for an application's main render/input loop to **also** read
raw controller state directly (`sceCtrlPeekBufferPositive` — see
[VitaSDK: system libraries](../02-vitasdk/05-system-libraries.md)) for its *own* navigation needs
(list scrolling, top-level mode switching between major screens) **independent of** whatever
`ImGuiIO.NavInputs`/focus-navigation state the backend is also feeding from the same physical
buttons. These two input paths — raw `sceCtrl` reads in your own code, and ImGui's own nav-input
system — can coexist deliberately in the same app, each owning a different layer of the UI (say,
top-level screen/tab switching handled by your own code, in-screen widget focus/activation handled
by ImGui's nav system). **Don't assume enabling gamepad navigation means ImGui now owns all input
handling in the app** — it's entirely normal, and often the right design, for the two to split
responsibility rather than one fully superseding the other.

## Touch: front vs rear, direct vs indirect

Front and rear touch panels are handled with independent enable flags in `imgui_vita_touch.cpp`
(`SCE_TOUCH_PORT_FRONT`/`_BACK`). An additional **"indirect" mode**
(`ImGui_ImplVitaGL_UseIndirectFrontTouch`) exists specifically for front-panel touch: instead of
jumping a virtual pointer straight to wherever a finger lands (direct/"absolute" mapping, natural
for a real touchscreen where you're tapping where you look), indirect mode **drags a persistent
virtual pointer relative to finger movement** — closer to how a laptop trackpad behaves than a
touchscreen. This distinction matters specifically for front touch because jumping a pointer
underneath your own finger the instant you touch down (rather than where you were already looking)
can feel disorienting in a way that doesn't come up the same way for rear touch (where there's no
visual surface behind your finger to create that expectation mismatch in the first place).

## No keyboard/text input

Covered in [The imgui-vita backend](02-imgui-vita-backend.md) and worth repeating here in the input-
handling context: there is no synthesized on-screen keyboard feeding ImGui's text-input widgets on
this platform. Any text-entry need in your UI has to be implemented as your own integration with
Sony's native `sceImeDialog` (open it, let the system render and handle the modal keyboard overlay,
read the result back), invoked from your own code at the point a text field would otherwise want
focus — not something the ImGui backend does for you automatically the way a desktop backend's
keyboard handling would.

## Practical guidance

- Decide your app's control scheme deliberately (gamepad-nav-first vs pointer-emulation-first vs
  touch-first) rather than leaving all three sources on by default.
- If going gamepad-nav-first, disable stick-as-mouse and set `MouseDrawCursor = false` explicitly —
  leaving stray pointer/cursor behavior active in a controller-first UI reads as an unfinished
  integration.
- It's fine, and often correct, for your own render-loop code to read raw controller state directly
  for things outside ImGui's own widget focus system — the two don't need to be mutually exclusive.
- Plan text-entry UX around `sceImeDialog` explicitly from the start; it's not something you can bolt
  on as an afterthought once you discover ImGui has no built-in answer for it on this platform.
