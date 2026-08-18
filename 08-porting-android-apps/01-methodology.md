# Loading a native `.so` and faking its JNI surface

## soloader-boilerplate — loading & running the native library

**[soloader-boilerplate](https://github.com/v-atamanenko/soloader-boilerplate)** is a fork-and-adapt
template (not a drop-in library) for this exact task. Its own framing: the same broad approach as
Wine — intercept the OS calls a binary expects rather than emulate the whole originating platform.
Concretely, for a `.so`:

1. **Load & relocate** — the `.so` is loaded into memory and its relocations applied, since it
   wasn't linked expecting to run at whatever address Vita's loader actually places it at.
2. **Symbol resolution** — imports the `.so` expects from Android's Bionic libc/NDK are mapped to
   Vita-native implementations via a symbol table you provide.
3. **JVM shim** — a fake JNI environment (see FalsoJNI below) handles the Java-side calls the `.so`
   makes back out.
4. **Syscall remapping** — Android NDK-specific calls get reimplemented against Vita equivalents.

As a template, you fork it and edit four files for a specific port: `CMakeLists.txt` (app metadata,
data paths), `dynlib.c` (map the `.so`'s unresolved symbols to Vita implementations), `java.c`
(implement the specific JNI methods/fields *this* `.so` actually calls), and `main.c` (input/render
loop wiring). Its own README doesn't enumerate limitations explicitly, but the shape of the problem
implies real ones: expect struct-layout mismatches between Bionic and Vita's own libc to surface as
you go, a dependency on kernel-level plugins (`kubridge`) for some of the lower-level shimming, and
real reverse-engineering work identifying exactly which JNI methods/fields a specific target `.so`
needs — `java.c` isn't something you can write generically ahead of knowing the target.

## FalsoJNI — faking the JNI/JVM surface

**[FalsoJNI](https://github.com/v-atamanenko/FalsoJNI)** (same author as soloader-boilerplate,
designed to pair with it) is a "zero-dependency fake JVM/JNI interface written in C" — it provides
the `JNIEnv`/`JavaVM` objects and intercepts the JNI call surface (`GetMethodID`,
`CallVoidMethodV`, etc.) a native library expects, without an actual JVM anywhere. You map specific
Java method/field names to real C implementations through configuration arrays, establishing the
correspondence the native code needs; Java arrays get represented as a `JavaDynArray` struct that
preserves size information, and Java type signatures get converted to their C equivalents.
Bootstrapping a port means initializing FalsoJNI, loading the native library's symbols, then
calling its `JNI_OnLoad` with the fake `JavaVM` — from the native code's perspective, that's
indistinguishable from a real JVM handing it a bootstrap call.

**Explicitly unimplemented per its own TODO list**: exception handling ("completely ignored"),
reference tracking, monitors, and direct byte buffers. If the specific `.so` you're porting relies
on any of those JNI features, budget time to either implement the missing piece yourself or confirm
the native code's actual code paths don't exercise it.
