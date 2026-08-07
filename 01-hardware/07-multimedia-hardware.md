# Multimedia Hardware

## Hardware video decode: H.264 limits

The Vita has a dedicated hardware H.264 decoder block (`SceVideodec`/`SceAvcdec` at the low level,
usually accessed indirectly through the higher-level `SceAvPlayer` — see below). Real hardware
enforces real decoder limits that the *emulator* (Vita3K) does not enforce with the same strictness
— this is a genuinely important, easy-to-miss gap between developing/testing on an emulator and
verifying on real hardware:

- Supported profiles: **Baseline / Main / High**.
- Supported level ceiling: **Level 3.1**. Level 3.1's `MaxFS` (max macroblocks per frame) is 3600,
  which works out to exactly `1280×720 / (16×16)` — i.e. **720p is the hardware ceiling**, at
  whatever frame rate the level's macroblock-processing-rate budget allows at that resolution.
  Content encoded at a *higher* level (Level 4.0, commonly what generic encoding presets like
  "HandBrake defaults" reach for even at 720p or lower, since encoders pick a level based on
  multiple parameters — reference frame count, DPB size, bitrate — not resolution alone) will be
  rejected or silently fail to decode on real hardware, while Vita3K's software decoder will happily
  decode it anyway.
- **Practical implication**: if you're shipping H.264 content for playback via `SceAvPlayer`,
  re-encode explicitly targeting Level 3.1 or lower and verify with `ffprobe` before shipping —
  don't trust "it played fine in the emulator" as proof it'll work on a real console. A concrete
  `ffmpeg` recipe: `ffmpeg -i in.mp4 -c:v libx264 -profile:v main -level 3.1 -vf scale=960:544 -c:a
  aac out.mp4` (scaling to the Vita's native 960×544 display resolution is also worth doing
  regardless of level, since decoding a larger frame than what's ever displayed wastes decode-time
  and memory for no visual benefit).

## SceAvPlayer

`SceAvPlayer` is Sony's higher-level audio/video playback API — the one most homebrew actually uses
rather than driving the raw `SceAvcdec`/`SceVideodec` decoder APIs directly. Key structural pieces:

- **`SceAvPlayerInitData`** — the init struct: memory-callback replacement, file-I/O-callback
  replacement (for streamed/custom-sourced content instead of a plain local path), event-callback
  replacement, base thread priority, number of output video frame buffers, autostart flag.
- **`SceAvPlayerMemReplacement`** — four callbacks: `allocate`/`deallocate` (generic working memory)
  and `allocateTexture`/`deallocateTexture` (**video frame memory specifically** — this is the hook
  point that determines which memory pool decoded frames actually live in; see
  [Memory architecture](04-memory-architecture.md) and
  [vitaGL: memory pools deep dive](../03-vitagl/06-memory-pools-deep-dive.md) — decoded video
  frames commonly need to come from the physically-contiguous pool).
- **Lifecycle events** delivered via the event callback (`SceAvPlayerEventCallback`, signature
  `(void *p, int32_t eventId, int32_t sourceId, void *eventData)`) — the event ID values themselves
  are **not part of the officially documented public header** (they're a community-reverse-engineered
  convention, most commonly seen as `SCE_AVPLAYER_STATE_READY == 2`, used to gate a
  `sceAvPlayerEnableStream()` call per discovered stream before starting playback). Treat any
  specific numeric event-ID meaning you find in community code as convention, not guaranteed Sony
  documentation, and verify against real-hardware behavior for your own use case rather than
  assuming it's universally reliable.
- **Known real-hardware gotcha**: `sceAvPlayerClose()` does not reliably invoke the
  `deallocateTexture` callback for whatever buffer the player was actively using at the moment of
  close. Code that trusts the callback contract alone to release everything can leak
  physically-contiguous memory across repeated open/close cycles. The robust pattern is to track
  every buffer your own `allocateTexture` callback hands out and explicitly free anything still
  outstanding after `sceAvPlayerClose()` returns, rather than assuming the SDK already did it.

### `sceAvPlayerEnableStream`/`sceAvPlayerStreamCount` aren't officially documented at all

The official reference (`docs.vitasdk.org/group__SceAvPlayerUser.html`) documents a minimal surface
— `Init`, `AddSource`, `Start`/`Stop`/`Pause`/`Resume`, `GetAudioData`/`GetVideoData`,
`GetStreamInfo`, `IsActive`, `SetLooping`, `SetTrickSpeed`, `CurrentTime`, `JumpToTime`, `Close` —
with **no** `EnableStream`, **no** `StreamCount`, and no documented event-ID enum at all. The
`StreamCount → GetStreamInfo → EnableStream → Start` pattern seen in a lot of community sample code
(gated on an undocumented `SCE_AVPLAYER_STATE_READY == 2` event ID) is real, working convention in
*some* codebases, but it is not Sony's documented contract, and — confirmed through direct,
systematic testing on real hardware — it is not a required step: `AddSource` with `autoStart` set
and no event callback at all is the complete, documented flow, and is exactly what at least one
independent, actively-distributed local-file MP4 player homebrew project (`vita-sample-avplayer`)
uses successfully.

### A real, hardware-confirmed limitation: local playback can fail categorically, independent of any application code

On at least one real console/firmware combination, local (non-streamed) video playback through the
stock `SceAvPlayer` module was confirmed, via a from-scratch minimal isolation test with zero
shared code against a larger application, to **never produce a single decoded frame** — regardless
of video encoding (profile/level/resolution), container structure (`moov` atom position/faststart),
memory pool sizing, output buffer count, `EnableStream` usage or omission, event-callback presence,
or player-handle lifecycle (fresh `Init` per open vs. a long-lived handle reused via `AddSource`/
`Stop`). `sceAvPlayerAddSource` reports success, `sceAvPlayerIsActive` intermittently reports true,
`allocateTexture` callback allocations succeed — but `sceAvPlayerGetVideoData` never yields a frame,
even after 3000+ polls over roughly a minute of real time.

The one working reference implementation found with confirmed local playback
(`SonicMastr/Vita-Media-Player`) depends on loading a companion **kernel-mode** module
(`SonicMastr/ReAvPlayer`, `reAvPlayer.suprx`) as a hard prerequisite before touching `SceAvPlayer`
at all — a `taiHEN`-based patch hooking several `SceAvcodecUser` decode-path functions and
binary-patching the loaded `SceAvPlayer` module in memory at a fixed, firmware-build-specific
offset. Its own README describes it as "NOT for normal user use, a developer tool," and it's
firmware-version-fragile by construction (a hardcoded patch offset). If you hit this exact
category of failure — local decode silently never starting, isolated from every application-level
variable — a kernel-level patch dependency (this one or an equivalent) may be the actual missing
piece, not a bug in your own init/call sequence. Treat adopting such a dependency as a deliberate,
**optional** choice (attempt to load it, gracefully degrade if unavailable), not a hard requirement
— the risk profile (kernel-level, firmware-specific, explicitly labeled non-production) is
disproportionate to make mandatory for a whole application over one feature.

## Audio hardware and SceAudio

Audio output goes through `SceAudioOut` (raw PCM output ports, `sceAudioOutOpenPort`/
`sceAudioOutOutput`) at the lowest level most homebrew touches directly, with higher-level codec
libraries (Vorbis, Opus, MP3-family, tracker-module formats via `libxmp`, etc. — commonly linked as
static libraries against VitaSDK, since the platform doesn't ship these as system codecs the way
video decode is a system-provided hardware block) doing the actual decode work in software on the
CPU before handing PCM to the audio-out port. There's no dedicated hardware audio *decode* block
analogous to the H.264 decoder — audio codec support is a software/library concern, not a hardware
capability question, unlike video.

## Practical takeaways

- Always verify media playback on **real hardware**, not just Vita3K — H.264 level/profile
  enforcement is one of the sharpest emulator-vs-hardware behavioral gaps on this platform.
- Track and explicitly release anything allocated through a media-playback memory callback yourself;
  don't assume the SDK's own close/teardown path reliably calls back through your deallocator on
  every exit path.
- Event-ID constants for `SceAvPlayer`'s callback are community convention, not official Sony
  documentation — validate them empirically for your own use case.
