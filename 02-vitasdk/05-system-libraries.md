# System Libraries

A tour of the higher-level `sce*` system libraries most homebrew ends up touching beyond the core
kernel/IO surface covered in [Kernel/core APIs](04-kernel-core-apis.md).

## Display

**`sceDisplay`** — low-level framebuffer/display control (setting the active framebuffer,
vsync/frame-count queries). Most graphics-heavy homebrew doesn't call this directly at all — it sits
underneath vitaGL (or raw sceGxm), which owns the actual display-buffer swap chain. Worth knowing it
exists mainly for understanding what layer vitaGL's `vglSwapBuffers` is ultimately built on.

## Input

- **`sceCtrl`** — buttons/analog sticks, see
  [Hardware: input devices](../01-hardware/06-input-devices.md).
- **`sceTouch`** — front/rear touch panels, independently queryable per port.
- **`sceMotion`** — accelerometer/gyroscope, requires explicit sampling start.
- **`sceCamera`** — front/rear camera capture.

All of the above should be treated as *optional, queryable* capabilities where hardware variance
exists (rear touch and motion are absent on PSTV) — see
[Hardware: input devices](../01-hardware/06-input-devices.md) for the compatibility angle.

## Audio

- **`sceAudioOut`** — raw PCM output ports (`sceAudioOutOpenPort`, `sceAudioOutOutput`,
  `sceAudioOutSetConfig`). Codec decoding (Vorbis, Opus, MP3-family, tracker formats) is a
  software/library concern layered on top, not a system-provided service — see
  [Hardware: multimedia hardware](../01-hardware/07-multimedia-hardware.md).
- **`sceAvPlayer`** — the higher-level audio/video playback API, covered in depth in
  [Hardware: multimedia hardware](../01-hardware/07-multimedia-hardware.md) since most of what's
  interesting about it is really about the hardware decoder it drives.

## Dialogs

**`sceCommonDialog`** (and its more specific siblings — `sceImeDialog` for text entry,
`sceMsgDialog` for message boxes, save-data dialogs, etc.) is Sony's native modal-UI-overlay system.
The pattern is consistent across all of them: initialize a params struct, call the dialog's `open`
function, poll its status each frame (`sceXxxDialogGetStatus`) while still driving your own render
loop underneath it (dialogs are typically rendered *by the system*, composited over whatever your
app is doing, not something that pauses your app entirely), then read the result once it reports
finished, and close it. This matters most concretely for **text input**: there is no first-party
on-screen-keyboard widget in most third-party UI toolkits used on this platform (Dear ImGui included
— see [imgui-vita](../04-imgui/README.md)), so text entry idiomatically goes through
`sceImeDialog` rather than a custom-built on-screen keyboard.

## App management

- **`sceAppMgr`** — launching other apps (`sceAppMgrLaunchAppByName`, keyed by title ID — see
  [Homebrew app anatomy](03-homebrew-app-anatomy.md)), querying the current app's own info, and
  related process-management concerns.
- **`sceAppUtil`** — app-level init/boot-parameter handling (`sceAppUtilInit`,
  `sceAppMgrGetAppParam` for reading how the app was launched — e.g. was it launched with specific
  boot parameters for an on-demand update flow).
- **`scePromoterUtil`** — registering an extracted package as a properly-installed, launchable app
  (used both by system installers and by homebrew self-update flows — see
  [Homebrew app anatomy](03-homebrew-app-anatomy.md)).

## Networking

**`sceNet`**/**`sceNetCtl`**/**`sceHttp`**/**`sceSsl`** — see
[Hardware: networking](../01-hardware/08-networking.md) for the full picture, including why most
homebrew reaches for a ported `libcurl` instead of these directly.

## A general pattern worth internalizing

Most of these libraries follow the same rough shape: **load the sysmodule if it's not
auto-loaded** (`sceSysmoduleLoadModule`), **initialize** with a params struct, **poll or callback**
for asynchronous results rather than blocking synchronously (this platform leans heavily on
poll-per-frame and callback patterns rather than blocking calls, which fits naturally with how a
game/UI render loop is structured anyway), and **explicitly tear down/release** when done. Once
that shape is familiar from one library, the others tend to follow it closely enough that reading
new API surface goes quickly.
