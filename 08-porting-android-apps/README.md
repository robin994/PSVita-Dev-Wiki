# Porting Android apps/games

A third porting shape, distinct from the other two this wiki covers: [porting a single OpenGL-based
library](../06-porting-opengl-libraries-to-vitagl/README.md) brings existing *source* onto vitaGL,
and [porting a decompiled console game](../07-porting-decompiled-games/README.md) brings a whole
decompiled game's *source* over via the libultraship/Shipwright pattern. This section covers a
different starting point entirely: an **Android APK you don't have (and don't need) the source
for** — running its compiled native library directly on Vita hardware, the same broad idea as
running a Windows binary under Wine rather than recompiling it.

## Why this works at all

Vita's CPU (ARMv7, Cortex-A9) is the same architecture family Android devices used from roughly
2012–2019. An Android game's actual game logic almost always lives in a compiled native library
(`.so`, built against Android's Bionic libc and NDK) that the Java/Kotlin app layer just loads and
calls into via JNI — extract that `.so` and hand it real ARMv7 code to run, and the CPU doesn't care
that it wasn't originally built for this platform. What it *does* need is everything that `.so`
expected to find at runtime: JNI/JVM calls it made into the Java side, and libc/NDK symbols it
expected Bionic to provide.

## The two pieces

1. **[Loading & running the native library](01-methodology.md)** — extracting the `.so`, applying
   relocations, and shimming the Android-specific syscalls/libc surface it expects.
2. **Faking the JNI/JVM surface** the `.so` calls back into, covered in the same page — this is
   what [FalsoJNI](01-methodology.md#falsojni--faking-the-jnijvm-surface) specifically solves.

Both pieces are typically used together, not as separate standalone choices — a native Android
`.so` almost always needs both a loader/relocator *and* a JNI shim to actually run, since the two
problems (get the code executing at all, satisfy the calls it makes back into "Java") are
inseparable in practice.

## Scope and honesty about sourcing

Neither tool on this page has been used hands-on for a real port in this wiki's own project work —
this section is sourced from each project's own README/documentation, not from independent
verification the way [vitaGL's](../03-vitagl/README.md) or [vita2d's](../05-vita2d/README.md)
sections are backed by real findings. Treat the architectural shape described here as accurate to
what the tools claim, and verify specifics against the actual source before depending on them for a
real port.
