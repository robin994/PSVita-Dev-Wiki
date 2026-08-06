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
