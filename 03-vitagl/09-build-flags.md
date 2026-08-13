# vitaGL: Build-Time Flags Reference

vitaGL ships as source you compile yourself (`make install`, from the vitaGL repo root) — most of
its behavior variants are chosen at **compile time** via `make FLAG=1 install`, not at runtime.
There is no single "default" build that fits every project: which flags a given prebuilt
`libvitaGL.a` was actually built with is a fact about *that specific binary*, not something you can
infer from the header alone (`vitaGL.h` doesn't change based on which of these were set). See
[Common pitfalls](08-common-pitfalls.md) for what goes wrong when this gets lost track of.

## How flags are actually set

```sh
# from the vitaGL repo root
make PHYCONT_ON_DEMAND=1 HAVE_CUSTOM_HEAP=1 install
```

Each `FLAG=1` on the command line becomes a `-DFLAG` (or a small fixed set of `-D`s, for flags that
expand to more than one define — see the `PHYCONT_ON_DEMAND` and `HAVE_DEBUGGER=2`-style multi-define
cases below) added to the library's own internal `CFLAGS`, then the whole library — every `.c`
translation unit — is rebuilt with that flag active. `install` copies the resulting `libvitaGL.a` and
`vitaGL.h` into `$VITASDK/arm-vita-eabi/{lib,include}`, so a project's own `CMakeLists.txt`/`Makefile`
just links against `-lvitaGL` normally afterward — nothing in the consuming project's own build needs
to change to pick these up.

**Consequence worth internalizing**: a flag like `PHYCONT_ON_DEMAND` only changes behavior if it was
defined when *vitaGL itself* was compiled — setting it in your own app's `CFLAGS` does nothing, since
none of your own translation units contain the `#ifdef PHYCONT_ON_DEMAND` branches; those live inside
vitaGL's source. Getting this flag applied means rebuilding (or obtaining an already-rebuilt copy of)
`libvitaGL.a` itself, not adding a define to your app's compile line.

## Debug flags

| Flag | Effect |
| --- | --- |
| `HAVE_SHARK_LOG=1` | Enables logging support in the runtime shader compiler. |
| `LOG_ERRORS=1` | Logs errors via `sceClibPrintf`. |
| `LOG_ERRORS=2` | Same, plus `-DFILE_LOG` (errors logged to a file instead of/alongside console). |
| `HAVE_PROFILING=1` | Lightweight profiler for CPU time spent in draw calls. |
| `HAVE_DEBUGGER=1` | Lightweight on-screen debugger interface. |
| `HAVE_DEBUGGER=2` | Same, plus extra info — **devkit only** (`-DHAVE_DEVKIT -DHAVE_RAZOR -DHAVE_DEBUG_INTERFACE`). |
| `HAVE_RAZOR=1` | Debugging via Razor debugger (retail + devkit compatible). |
| `HAVE_DEVKIT=1` | Extra debugging via Razor, devkit-only (`-DHAVE_DEVKIT -DHAVE_RAZOR`). |
| `HAVE_CPU_TRACER=1` | Inserts a Razor CPU sync at every buffer swap for better profiler frame timelines. |
| `DEBUG_GLSL_TRANSLATOR=1` | Logs GLSL translator input/output before shader compilation. |
| `DEBUG_GLSL_PREPROCESSOR=1` | Logs GLSL preprocessor input/output before shader compilation. |
| `DEBUG_GC=1` | Sanity checks for the internal garbage collector. |
| `DEBUG_THREAD_SAFENESS=1` | Sanity checks for thread-safety usage of the library. |

## Compatibility flags

| Flag | Effect |
| --- | --- |
| `HAVE_CUSTOM_HEAP=1` | Replaces the `sceClibMspace` heap implementation with a custom one — safer, with proper diagnostics. Changes how *every* pool (`VGL_MEM_RAM`/`VGL_MEM_VRAM`/`VGL_MEM_BUDGET`, and `VGL_MEM_SLOW` when `PHYCONT_ON_DEMAND` is *not* also set) allocates internally — see [Memory pools deep dive](06-memory-pools-deep-dive.md). |
| `HAVE_GLSL_TEXTURE_SIZE=1` | Experimental automatic handling of GLSL `textureSize` calls via the GLSL translator. |
| `HAVE_GLSL_UBOS=1` | Experimental support for nominal uniform buffer objects in GLSL shaders. |
| `HAVE_FFP_SHADER_SUPPORT=1` | Support for GLSL 1.20 legacy fixed-function-pipeline uniform bindings (e.g. `gl_ModelViewProjectionMatrix`). Slightly slower shader pipeline. |
| `SOFTFP_ABI=1` | Compiles the library in soft-float ABI compatibility mode (`-mfloat-abi=softfp -DHAVE_SOFTFP_ABI`). |
| `STORE_DEPTH_STENCIL=1` | Framebuffer depth/stencil surfaces get loaded/stored to memory — slower, more OpenGL-standard-compliant. |
| `HAVE_HIGH_FFP_TEXUNITS=1` | Support for more than 2 texture units in the fixed-function pipeline, at some performance cost. |
| `HAVE_DISPLAY_LISTS=1` | Support for display lists, at some performance cost. |
| `SAFE_ETC1=1` | Disables hardware ETC1 texture support — less efficient, but debuggable in Razor (`-DDISABLE_HW_ETC1`). |
| `SAFE_DRAW=1` | Less-optimized draw pipeline; can fix some rendering glitches (`-DSTRICT_DRAW_COMPLIANCE`). |
| `SAFE_UNIFORMS=1` | Less-optimized shader uniform pipeline; makes basic-type-array uniform location indexing spec-compliant (`-DSTRICT_UNIFORMS_COMPLIANCE`). |
| `UNPURE_TEXFORMATS=1` | Support for texture dimensions other than 2D (`tex2D` still required in shader code). |
| `ENABLE_LEGACY_PIPELINE=1` | Support for the legacy `vglDrawObjects` pipeline — relevant to the argument-count drift covered in [Common pitfalls](08-common-pitfalls.md). |
| `HAVE_FIXED_ATTRIBUTES=1` | Experimental `GL_FIXED` attribute support in GLSL-shader codepaths. |

## Hack / speedhack flags (all trade correctness or stability for speed)

| Flag | Effect |
| --- | --- |
| `NO_TEX_COMBINER=1` | Disables `GL_COMBINE` texture-combiner support for faster fixed-function draws (`-DDISABLE_TEXTURE_COMBINER`). |
| `NO_DEBUG=1` | Disables most error handling (`-DSKIP_ERROR_HANDLING`) — faster, less OpenGL-standard-compliant. |
| `BUFFERS_SPEEDHACK=1` | Faster vertex-buffer copying. **May cause crashes.** |
| `DRAW_SPEEDHACK=1` | Faster draw-call code. **May cause crashes.** |
| `DRAW_SPEEDHACK=2` | Faster draw-call code, only for large vertex-data draws (`-DSAFER_DRAW_SPEEDHACK`). **May cause crashes.** |
| `INDICES_DRAW_SPEEDHACK=1` | Faster index-buffer handling for draws. **May cause crashes.** |
| `INDICES_SPEEDHACK=1` | Faster draw code; disables instanced-draw support, 32-bit (`GL_UNSIGNED_INT`) indexed draws may glitch. |
| `MATH_SPEEDHACK=1` | Faster matrix math. **May cause glitches.** |
| `TEXTURES_SPEEDHACK=1` | Non-fully-compliant `glTexSubImage2D`/`glTexSubImage1D`, faster rendering. Incompatible with `HAVE_TEXTURE_CACHE=1`. |
| `TEXTURE_UPLOADS_SPEEDHACK=1` | Faster PO2 compressed-texture uploads. **Might cause crashes.** |
| `SAMPLERS_SPEEDHACK=1` | Faster sampler resolution during shader use. **May cause glitches.** |
| `PRIMITIVES_SPEEDHACK=1` | More efficient draw calls; `GL_LINES`/`GL_POINTS` usage may glitch. |
| `DEPTH_STENCIL_HACK=1` | Zero memory cost for depth/stencil buffers. **Can cause crashes** in some circumstances. |
| `READBACKS_SPEEDHACK=1` | Faster `glReadPixels`. **May cause glitched readbacks.** |
| `CIRCULAR_POOL_SPEEDHACK=1` | Internal circular pool uses a single buffer — slight CPU win, may glitch. |

Given how many of these are explicitly "may cause crashes/glitches," treat the hack-flag group as
something to reach for deliberately on a specific, measured performance problem — not as a default
"turn them all on for speed" bundle.

## Misc flags

| Flag | Effect |
| --- | --- |
| `HAVE_TEXTURE_CACHE=1` | File-caching for long-unused textures — a swap-like mechanism to extend effective available memory. Experimental (`-DHAVE_TEX_CACHE`). |
| `NO_DMAC=1` | Disables `sceDmacMemcpy` usage — can improve framerate in rare cases. |
| `HAVE_UNFLIPPED_FBOS=1` | Framebuffer objects are not internally flipped to match OpenGL convention. |
| `HAVE_WVP_ON_GPU=1` | Moves fixed-function-pipeline world-view-projection calculation to the GPU — less CPU load, more GPU load. |
| `SHARED_RENDERTARGETS=1` | Small framebuffer objects share rendertargets instead of getting dedicated ones. |
| `SHARED_RENDERTARGETS=2` | Same, plus older-rendertarget recycling (`-DHAVE_SHARED_RENDERTARGETS -DRECYCLE_RENDERTARGETS`). |
| `NO_CIRCULAR_POOL=1` | Temporary data buffers skip the circular pool — less memory, worse CPU performance (`-DDISABLE_CIRCULAR_POOL`). |
| `USE_SCRATCH_MEMORY=1` | `GL_DYNAMIC`/`GL_STREAM` VBOs can use the circular pool instead of regular allocations. Incompatible with `NO_CIRCULAR_POOL`. |
| `HAVE_PTHREAD=1` | Uses `pthread` instead of `sceKernel` to start the garbage-collector thread. |
| `SINGLE_THREADED_GC=1` | Garbage collector runs on the main thread instead of its own. |
| **`PHYCONT_ON_DEMAND=1`** | **Makes physically-contiguous RAM (`VGL_MEM_SLOW`) be handed out as separate, individually-allocated memblocks (`sceKernelAllocMemBlock` per request) instead of being carved from one heap set up at init.** This is the flag [`SceAvPlayer`'s video-frame memory issue](../01-hardware/07-multimedia-hardware.md) is really about — without it, `vglAlloc(_, VGL_MEM_SLOW)` sub-allocates from a shared heap-managed pool, and consumers that specifically require a dedicated, directly-mapped memblock (rather than any address that merely falls within a larger pool) don't get one, even though the allocation call itself reports success. See the "Verifying a substitute vitaGL build" section below before assuming a rebuild with this flag alone fixes such an issue. |
| `UNPURE_TEXTURES=1` | Legal to upload textures without a base level. |
| `UNPURE_TEXCOORDS=1` | Legal to use multitexturing in the fixed-function pipeline with `GL_TEXTURE0` disabled. |
| `DISABLE_FFP_MULTITEXTURE=1` | Disables multitexture processing in fixed-function-pipeline draw calls. |
| `HAVE_WRAPPED_ALLOCATORS=1` | Allows vitaGL's own allocators to be used inside `--wrap`-wrapped `malloc`/`free`/etc. Only relevant if your *app's* link step uses `-Wl,--wrap=malloc` (or similar) itself — most projects don't. |
| `HAVE_SHADER_CACHE=1` | Fast automatic file caching (XH3/xxHash-based) for app-provided shaders. |
| `NO_CLIB=1` | Disables `sceClib*` function usage for easier debugging, slightly slower CPU code. |
| `DISABLE_W_CLAMPING=1` | Disables W-clamping during viewport calculation — might fix some glitches. |
| `NO_TILE_CLIPPER=1` | Disables early tile clipping for scissor testing — less CPU work, more GPU work. |
| `NO_SPLASHSCREEN=1` | Disables vitaGL's boot splashscreen (which otherwise hides loading time) — `-DSKIP_SPLASHSCREEN`. |

## Verifying a substitute vitaGL build actually renders

A real, hard-won lesson from this project: **a `libvitaGL.a` you cannot see the exact source/flags
for is not something you should assume is equivalent to another `libvitaGL.a` just because the
`vitaGL.h` header is byte-identical between them.** Two vendored copies with an identical header —
different build (different flags, possibly a different upstream commit despite "looking like the
same vintage") — can still produce a binary that compiles and links cleanly against your app but
**does not render anything on real hardware** (a black screen from boot, unrelated to any specific
feature you're trying to test).

If you ever need to rebuild vitaGL yourself to pick up one flag (rather than obtaining an
already-rebuilt library from whoever built the known-working one):

1. Rebuild the **whole library**, not a single hand-patched translation unit swapped into an existing
   archive. Compiling just the one `.c` file that contains the flag you care about and `ar`-replacing
   that one object in an otherwise-untouched archive risks a flags/ABI mismatch against the rest of
   the archive that can silently break unrelated functionality — general rendering included, not just
   the feature you meant to change.
2. Before trusting *any* flag-level change, first confirm the **unmodified** baseline you're building
   from actually renders on real hardware. If it doesn't, the flag isn't your problem yet — the base
   library choice is.
3. Test on real hardware, not just Vita3K, for anything touching memory-pool behavior specifically —
   see [Debugging & tooling: emulator vs real hardware](../02-vitasdk/07-debugging-tooling.md).
