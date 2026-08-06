# Input Devices

## Buttons and analog sticks

Standard face buttons (cross/circle/square/triangle), shoulder buttons (L/R, called "L1/R1" on
some later hardware with extra trigger-style buttons — check what your target range actually has),
D-pad, Start/Select, and two analog sticks. Read via `sceCtrlPeekBufferPositive` (non-blocking,
returns latest state) or `sceCtrlReadBufferPositive` (blocking until a new sample is available) —
`Peek` is what the overwhelming majority of homebrew render-loop input handling actually uses,
since you want "current state this frame," not "block until something changes."

Button state comes back as a bitmask (`SCE_CTRL_CROSS`, `SCE_CTRL_CIRCLE`, etc.) you `&` against;
analog stick axes come back as unsigned 8-bit values centered around 128 (i.e., you need to
subtract 128 and normalize/deadzone yourself — the raw API gives you no built-in deadzone handling).

**Note on face-button meaning**: circle-as-confirm/cross-as-cancel vs the opposite is a
region-dependent UX convention (Japan traditionally uses the opposite mapping from the West) that
Sony's own system UI respects via a system setting. Well-behaved homebrew UI that wants to match
platform convention should read that system setting rather than hardcoding one assignment — see the
[VitaSDK system libraries page](../02-vitasdk/05-system-libraries.md) for the relevant API.

## Touch

Two independent capacitive multi-touch panels: **front** (over the screen, `SCE_TOUCH_PORT_FRONT`)
and **rear** (`SCE_TOUCH_PORT_BACK`, the pad on the back of the unit — **absent on PSTV**, so code
that reads back-touch needs to handle that gracefully rather than assuming it's always present).
Queried via `sceTouchPeek`/`sceTouchRead` per port, returning up to several simultaneous touch
points with position and a rough pressure/force value.

Because the rear touchpad has no visual surface behind it, UI design for it is necessarily different
from front-touch UI — it's typically used for "grip" interactions (extra shoulder-trigger-like
inputs) rather than anything requiring visual point-and-tap precision.

## Motion sensors

Accelerometer and gyroscope (3-axis each), present on both handheld Vita revisions,
**absent on PSTV**. Accessed via `sceMotion*` APIs after enabling sampling
(`sceMotionStartSampling`). Used for tilt-based controls in games; rarely relevant to
utility/tool-style homebrew, but worth knowing exists if you're porting something that expects
motion input and need a PSTV-safe fallback path.

## Rear camera

Both handheld revisions have front and rear cameras; PSTV has none. Accessed via `sceCamera*` APIs.
Homebrew use is relatively rare outside of camera-specific apps/demos, but the same
"query capability, don't assume presence" principle applies if you ever touch it.

## On-screen keyboard / text input

There is **no hardware keyboard**, and most GUI toolkits used in homebrew (including Dear ImGui, as
ported for the Vita — see the [imgui-vita section](../04-imgui/README.md)) don't ship a built-in
soft-keyboard text-input widget of their own. Text entry on this platform idiomatically goes through
Sony's native **IME dialog** (`sceImeDialog`), a modal system UI overlay the app hands control to
temporarily and reads the result back from — not something you build yourself out of on-screen
button widgets, except in specialized cases. See
[VitaSDK: system libraries](../02-vitasdk/05-system-libraries.md) for the API shape.

## Designing for PSTV compatibility

If there's one input-related best practice worth internalizing: **treat rear touch, motion sensors,
and the camera as optional, queried capabilities, never as guaranteed-present hardware.** A
significant and still-active slice of the Vita homebrew audience runs on PSTV specifically (it's
popular for always-on "media center"-style homebrew use), and input code that assumes handheld
hardware will misbehave there in ways that are easy to avoid by simply checking before using.
