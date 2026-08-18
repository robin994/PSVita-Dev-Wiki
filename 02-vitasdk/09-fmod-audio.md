# FMOD Audio (FMOD-PSV)

FMOD is a proprietary cross-platform audio middleware (Firelight Technologies) extremely common in
Android game ports — many Unity and native-Android titles built their audio around it rather than
platform APIs. **[GrapheneCt/FMOD-PSV](https://github.com/GrapheneCt/FMOD-PSV)** is the community
bridge that makes a Vita build of FMOD's C API available to VitaSDK homebrew. It matters specifically
for the "porting an Android game/engine to Vita" case: if the source project's audio layer already
targets FMOD, this is the piece that lets you keep that layer largely as-is instead of rewriting
audio against `sceAudioOut` from scratch.

**Read this whole page before assuming it solves your problem** — the single most important fact
(see "Which FMOD generation" below) is that it only covers the *old* FMOD Ex/Designer C API, not
modern FMOD Studio, and most FMOD-using Android games from the last several years are on Studio.

## Doc-archaeology warning: the repo's current README says almost nothing

At time of writing, `README.md` in the repo is literally two words: `# FMOD-PSV`. Everything below
was reconstructed from the commit history (which held a much fuller README until it was wiped by
the final "New impl" commit), the `.vcxproj`/`.yml` build files still in the tree, and the repo's one
GitHub issue — not from current documentation. If you go looking at the repo yourself, don't stop at
the README; the useful information is in `libfmodex/libfmodex/lib_vitasdk.yml` and
`libfmodex.vcxproj`.

## What it actually is

FMOD Ex/Designer has never had its Vita backend source publicly released — Firelight never shipped
a public Vita port, and Sony's official Vita SDK was itself never publicly released either. What
exists on the Vita is a build Unity Technologies licensed and statically linked into
`PSP2Player_Mono.self`, the Mono player shipped as part of Unity Editor's (long-discontinued)
PS Vita export target — available only to developers with Sony's official Vita SDK/devkit access.
FMOD-PSV is not a from-scratch port; it's tooling to extract a working `fmodex.suprx` system module
from that Unity-licensed binary (or, in its later revision, to rebuild an equivalent module directly
against Sony's official FMOD static libraries) and expose its exports as a `.suprx` any VitaSDK
homebrew can link against and load.

The repo went through two genuinely different implementations, and the current tree only reflects
the second one:

### Original approach (2022): patch Unity's bundled FMOD binary

Documented in early README revisions, since deleted:

1. Download `UnitySetup-Playstation-Vita-Support-for-Editor-2018.3.0a2.exe` (Unity Editor's Vita
   export-target installer — requires prior Vita devkit/NDA access to obtain; Unity has since
   discontinued the Vita export target entirely, so this installer is no longer distributed by
   Unity at all) and extract `PSP2Player_Mono.self` from it.
2. "Unself" it (strip Sony's SELF signing wrapper down to a plain ELF) and apply a binary patch via
   `xdelta3`, using a `.delta` file the repo used to ship (no longer present in the current tree).
3. Re-`fself` the patched ELF, preferably with strip and compress enabled.
4. Rename the result to `fmodex.suprx` — ready to load as a system module.
5. Optionally copy `fmodngp.h` (a Vita-specific header, also since removed from the repo) alongside
   your regular FMOD headers, for setting Vita-specific properties at `FMOD_System_Init` time.

**Known gotcha, confirmed by the repo's only issue** ([#1](https://github.com/GrapheneCt/FMOD-PSV/issues/1),
closed with no resolution posted): a user hit `xdelta3: target window checksum mismatch:
XD3_INVALID_INPUT` trying to apply the patch. That error means the `.delta` was built against a
specific byte-for-byte build of `PSP2Player_Mono.self` and the extracted input didn't match exactly
— almost certainly a different point-release of the Unity Vita-support installer than the one the
patch author used. Since the installer itself is no longer obtainable through official channels,
this whole path is largely dead for anyone starting from scratch today.

### Current approach (April 2023, "New impl" commit): rebuild against Sony's official FMOD libs

The tree today is just `libfmodex/libfmodex/` — a Visual Studio project (`libfmodex.vcxproj`,
`Debug|PSVita` / `Release|PSVita` configurations) built with **Sony's official PS Vita SDK
(VDSuite/PSVitaVSI, referenced via `$(SCE_PSP2_SDK_DIR)`), not VitaSDK/CMake**. `main.c` is 0 bytes
— there's no actual source in this project. It exists purely to link two prebuilt Firelight static
libraries (`-lfmodex -lfmodevent`, i.e. FMOD Ex core + the older FMOD Designer "Event" layer — not
included in the repo, sourced the same way as the Unity player above) against the Vita's audio/system
stack (`SceAudio`, `SceAudioIn`, `SceAudiodec`, `SceNgs` weak-linked, `SceNet` weak-linked,
`SceThreadmgr`, `SceSysmodule`, `SceFpu`) and produce a `.suprx`.

The interesting part is the post-build step, which chains three official-SDK host tools:

```
vdsuite-pubprx.exe --compress <self> <self>
armlibgen.exe --dump lib.emd --stub-archive
vdsuite-libgen.exe --output-kind VitasdkStub lib_vitasdk.yml vitasdk_stub
```

That last line is the actual point of the project: `vdsuite-libgen.exe` takes `lib_vitasdk.yml` (a
hand-authored NID database, module NID `0x959B78D0`) and emits a **VitaSDK-format import stub
library** — the piece that lets ordinary VitaSDK/CMake homebrew link against `fmodex.suprx` without
needing Sony's official SDK at all, *once a working `fmodex.suprx` exists on the device*.

## The critical catch: neither path is buildable from what's in the repo alone

Both implementations bottom out on assets GrapheneCt can't legally redistribute and doesn't include:

- **Path 1** needs Unity's now-delisted Vita-support installer, plus the `.delta` patch file the
  repo no longer ships.
- **Path 2** needs Sony's official PS Vita SDK (devkit-gated, never public) and Firelight's official
  `fmodex.lib`/`fmodevent.lib` static libraries for Vita.

What a VitaSDK-only developer without an official devkit or a Unity Vita license can actually get
from this repo today is `lib_vitasdk.yml` — the NID database describing what a working
`fmodex.suprx` exports, not a working module. In practice, using this depends on either building the
module yourself (if you have devkit/Unity access) or obtaining a prebuilt `fmodex.suprx` from
someone in the scene who already has. Treat this less as "grab a library and link" and more as
"a documented target to reproduce" — worth knowing about before you assume it's a drop-in dependency.

## Which FMOD generation this actually covers

Confirmed from both the pre-wipe README (`FMOD version is 4.44.56`) and by reading
`lib_vitasdk.yml`'s exported symbol list directly: this is the **FMOD Ex 4.4.x C API**, plus the
older **FMOD Designer "Event" layer** (`FMOD_Event*`, `FMOD_EventSystem_*`, `FMOD_EventGroup_*`,
`FMOD_EventCategory_*`, `FMOD_MusicSystem_*`, `FMOD_EventReverb_*` — 189 of the exported functions
belong to this layer alone), not modern FMOD Core/Studio (`fmod_studio.hpp`, the API most FMOD-using
Unity/Android titles from roughly the last decade actually use). The core-API surface covers the
expected object model: `FMOD_System_*`, `FMOD_Sound_*`, `FMOD_Channel_*`, `FMOD_ChannelGroup_*`,
`FMOD_DSP_*`/`FMOD_DSPConnection_*`, `FMOD_Reverb_*`, `FMOD_Geometry_*`, `FMOD_Memory_*`,
`FMOD_CODEC_*`.

**This is the fact to check first before investing in this bridge for a port.** A game built against
FMOD Studio's event-based runtime doesn't map onto this NID set at all — you'd be looking at either
re-targeting its audio layer to the Ex-era API (a real rewrite, not a relink) or concluding this
bridge doesn't apply to that project.

## Hardware backend

Per the pre-wipe README, FMOD's Vita output driver hooks Vita-native audio hardware rather than
going through a generic PCM path only:

- **`SceNgs`** — the Vita's native audio voice/mixing/DSP graph subsystem (not otherwise documented
  elsewhere in this wiki yet). Weak-linked in the current build (`-lSceNgs_stub_weak`), consistent
  with FMOD using it opportunistically for hardware-accelerated mixing/effects where available.
- **`SceAudiodec`** — the Vita's hardware audio-decode block, see
  [Hardware: multimedia hardware](../01-hardware/07-multimedia-hardware.md) for the general
  hardware-decoder picture (documented there for video/H.264; the audio decode block is the same
  family of dedicated hardware).

The README also claimed support for "Vita-specific formats and DSP effects," which isn't broken down
further anywhere in the repo — treat as an unverified vendor claim rather than a confirmed feature
list.

## Headers

Not included in the repo at any point in its current tree. You need your own FMOD Ex 4.4.x SDK
headers (the plain C API is largely platform-agnostic, so headers pulled from a PC/other-platform
FMOD Ex 4.4.x SDK download should line up), plus `fmodngp.h` — a Vita-specific header for setting
platform properties at `FMOD_System_Init` time that existed in early repo revisions and was removed
in the "New impl" rewrite along with everything else undocumented. You'd need to pull it from repo
history (`git log` on a shallow clone, or GitHub's commit view) rather than the current tree.

## Using it from a VitaSDK/CMake project

Once you have (a) a working `fmodex.suprx` on-device and (b) the generated VitaSDK stub library from
`lib_vitasdk.yml`:

```cmake
target_link_libraries(YourApp
  fmodex
  SceAudio_stub SceAudioIn_stub SceAudiodec_stub
  SceNgs_stub_weak SceNet_stub_weak
  SceThreadmgr_stub SceSysmodule_stub
  # ...your usual stack
)
```

(Library names above translate the `.vcxproj`'s official-SDK dependency list into VitaSDK's usual
`Sce*_stub`/`_stub_weak` naming convention — see
[Project setup & build system](02-project-setup-build-system.md) for that convention generally.)

`fmodex.suprx` itself has to actually be loaded before any `FMOD_*` call — it's not a system-bundled
module, so your app needs to ship it and load it at startup (the standard idiom elsewhere in this
ecosystem for a homebrew-supplied `.suprx`, e.g. vitaShaRK's shader-compiler module, is
`sceKernelLoadStartModule` against a path under your app's own data directory; the FMOD-PSV repo
doesn't document its own expected loading path, so this is inferred from the general pattern, not
confirmed against this specific module).

## Maintenance status

- Created July 2022, 18 commits total, last commit (the README-wiping "New impl" rewrite) April 11,
  2023. No commits since.
- 6 stars, 0 forks, 2 watchers as of this writing — small, low-activity project.
- No `LICENSE` file in the repo. Given that both build paths depend on Sony's official SDK and/or
  Firelight's proprietary static libraries obtained through a Unity licensing channel, treat the
  legal status of redistributing a built `fmodex.suprx` as genuinely unclear — this is scene tooling
  for personal/homebrew use, not a cleanly licensed open-source library.
- One issue on record ([#1](https://github.com/GrapheneCt/FMOD-PSV/issues/1), closed, no resolution
  comment) — the xdelta checksum-mismatch gotcha under the now-superseded original approach, noted
  above.

## Bottom line

FMOD-PSV is a real, working (per the linked NID database) bridge for **old-generation FMOD Ex/
Designer audio (4.4.x)** on Vita, hooking the console's native `SceNgs`/`SceAudiodec` hardware —
useful specifically when porting a game whose audio layer already targets that generation of FMOD.
But it's not a fetch-and-link dependency: neither documented build path ships everything needed to
produce a working module yourself, the module itself isn't included, and it's undocumented enough in
its current state that using it means reading `lib_vitasdk.yml` and the `.vcxproj` directly rather
than a README. Before adopting it for a specific port, confirm the source project is actually on
FMOD Ex/Designer and not modern FMOD Studio — that single check determines whether this bridge is
relevant at all.
