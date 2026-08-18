# Alternative audio APIs & non-native runtimes

Two different problems live on this page: porting audio code that already targets a standard API
other than `sceAudioOut`, and running an entire non-native runtime (Mono/.NET, PHP, a JS engine) on
top of Vita homebrew instead of writing C/C++ against VitaSDK directly. Grouped together because
each is "bring an existing standard interface to Vita rather than rewrite against the platform API,"
the same broader theme as [FMOD audio](09-fmod-audio.md).

## VitaAL — OpenAL 1.1 for Vita

**[vitaAL](https://github.com/GrapheneCt/vitaAL)** targets full OpenAL 1.1 compliance (with
possible EAX/SOFT extension support per its README), the same "keep an existing audio layer mostly
intact" value proposition as [FMOD-PSV](09-fmod-audio.md) — relevant when a port's source project
already targets OpenAL rather than needing an `sceAudioOut`-based rewrite. Documented limits: 8/16
bit mono/stereo up to 192 kHz, and a maximum of 4 buffers queued per source. The repo's own
documentation doesn't specify which underlying Vita audio subsystem it sits on ("hardware
accelerated" isn't broken down further than that one phrase) — verify against the actual source if
the specific `sceAudioOut`-vs-something-else backing matters for your use case.

## VitaMonoLoader — Mono/.NET (Unity's scripting backend) on Vita

**[VitaMonoLoader](https://github.com/GrapheneCt/VitaMonoLoader)** is a standalone Mono loader,
primarily aimed at **Unity games using the Mono scripting backend** (compiling managed code to
AOT/ahead-of-time assemblies) rather than porting a game's C++ engine code directly. Marked
work-in-progress upstream, with concrete stated constraints: **Windows-only host** for the build
tooling, targets VDS/vitasdk toolchains specifically (see
[Alternative toolchains & deployment](11-alternative-toolchains-deployment.md#vdsuite--a-third-sdktoolchain-lineage)
for what VDS/VDSuite is), requires the app be compiled in ARM mode, and needs the CapUnlocker
plugin plus specific Unity support tooling installed alongside it. This is a different porting shape
entirely from [porting a decompiled console game](../07-porting-decompiled-games/README.md) — you're
running the original managed bytecode/AOT output rather than porting native source.

## pp++ (PHP-Player-plus-plus) — an embedded PHP interpreter

**[pp++](https://github.com/isage/pppp-vita)** embeds a PHP8 interpreter (a Vita-specific PHP8 port
is a separate dependency) with SDL2/SDL2_mixer/SDL2_image bindings, letting homebrew be written in
PHP against a `index.php` entry point rather than C/C++. Genuinely niche — worth knowing exists if
you ever encounter a PHP-based Vita homebrew and need to understand how it's even possible, not a
tool this wiki would recommend reaching for on a new project.

## TriGL — WebGL for SCE Trilithium

**[TriGL](https://github.com/GrapheneCt/TriGL)** implements WebGL for **SCE Trilithium**, Sony's
own JavaScript engine on Vita (distinct from any homebrew JS runtime) — ships as two plugin modules
(`liext.suprx` for core extensions, `webgl.suprx` for the WebGL surface itself). Notably, its own
documentation states installation requires decrypting and modifying the **Crunchyroll app's**
Trilithium framework — this isn't a standalone homebrew library so much as a hook into one specific
system app's bundled JS engine. WebGL extensions aren't supported. Relevant only if you're dealing
with Trilithium-hosted JS content specifically (e.g. legacy PS Vita app content that used it) — not
a general-purpose graphics API choice for a new homebrew project the way vitaGL or vita2d are.
