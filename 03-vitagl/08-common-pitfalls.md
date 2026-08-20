# vitaGL: Common Pitfalls

## Version drift breaking API compatibility silently

The single most impactful pitfall to know about: vitaGL's API has had genuine, backward-incompatible
changes across its development history — the clearest documented example being
**`vglDrawObjects`'s argument count** changing between a 2-argument form (`mode, count`) and a
3-argument form (`mode, count, implicit_wvp`) at different points. Because this is an arity change
on an otherwise-plausible function call, it doesn't reliably manifest as a hard compile error if you
adapt a call site to match whatever header version happens to be present — you can end up with code
that *compiles cleanly* against a newer/older vitaGL than the surrounding codebase was actually
written for, and produces a binary that builds successfully but **doesn't render anything on screen**
(or renders incorrectly) on real hardware.

**Mitigations:**
- Pin a specific, known-working vitaGL commit/version for a given project rather than always
  tracking the latest checkout, and document which one.
- If a build error mentions `vglDrawObjects` (or any vitaGL function) with an unexpected argument
  count, treat that as a strong signal you're pointed at the wrong vitaGL version for the codebase —
  fix the version mismatch (correct include/library paths), don't just edit the call site to match
  whatever header happens to be present, since that can silently produce a build that compiles but
  doesn't actually work correctly at runtime.
- When multiple vendored/local copies of vitaGL exist in a development environment (a common
  situation — e.g. a "known good, older" copy alongside a "matches current upstream" copy), verify
  which one a given project's build is actually picking up (check the effective include/library
  search paths a build invocation resolves to) rather than assuming — this is a very easy thing to
  get silently wrong when several similarly-named library copies exist on a dev machine.

## The `VGL_MEM_SLOW` vs `VGL_MEM_RAM` mixup

Covered in depth in [Memory pools deep dive](06-memory-pools-deep-dive.md) — worth repeating here as
a named pitfall: substituting the general RAM pool for the physically-contiguous pool for a resource
that genuinely needs physical contiguity (video decode buffers being the clearest example) is a real,
observed mistake, and it's dangerous specifically because it doesn't necessarily fail immediately or
obviously — it can compile, link, and appear to function in casual testing before manifesting as a
decode failure or corruption under more demanding conditions.

## `vglSwapBuffers`'s boolean argument is not vsync

Also worth naming explicitly as a pitfall in its own right (see
[Rendering pipeline](03-rendering-pipeline.md) for the full explanation): it's easy to assume a
boolean argument on a frame-end/buffer-swap call controls vsync, by analogy with other graphics
APIs where that's a common pattern. On vitaGL, it actually indicates whether a native
`sceCommonDialog` overlay is currently active. Getting this backwards doesn't necessarily crash
anything — it just means dialog/rendering interop doesn't behave as intended, which can be a subtle,
easy-to-miss bug rather than an obvious one.

## Treating vitaGL as spec-conformant desktop/ES OpenGL

vitaGL implements a *subset* with some genuinely non-standard extensions layered in (mapped-pointer
draw calls, the multi-pool memory model — see [Overview](01-overview.md)). Code ported from a real
desktop/ES OpenGL codebase will need real adaptation, not just a recompile — assuming full spec
conformance and being surprised when a less-common GL feature isn't implemented (or behaves subtly
differently) is a common early-adoption pitfall for anyone coming from more mainstream GL platforms.

## vitaGL's `mod()` translation doesn't match GLSL semantics

If your project uses vitaGL's built-in GLSL→Cg auto-translator (see
[Shaders](05-shaders.md#vitagls-own-glslcg-auto-translator-a-third-path)) rather than hand-written
Cg, be aware that at least some vitaGL versions translate GLSL's `mod()` as a naive
`#define mod(x,y) fmod(x,y)`. This is wrong on two counts: GLSL's `mod()` is defined as
`x - y*floor(x/y)` (result takes the sign of `y`), while C's `fmod()` takes the sign of `x` — these
genuinely disagree for negative operands — and the macro form also doesn't support GLSL's
guaranteed `mod(vecN, float)` scalar-second-argument overload, which `fmod` can't provide directly.
Any shader relying on `mod()`'s wraparound behavior with negative inputs (a very common pattern —
UV wrapping, phase/angle wrapping, dithering/noise patterns computed from screen coordinates) will
silently produce wrong output rather than fail to compile, since both `fmod` and the macro are
syntactically valid.

**Mitigations:**
- Replace the macro with real overloaded functions implementing the correct formula, one per
  argument-type combination GLSL supports (`float`/`float2`/`float3`/`float4`, and a scalar-`y`
  overload for each vector width):
  ```c
  inline float mod(float x, float y) { return x - y * floor(x / y); }
  inline float2 mod(float2 x, float y) { return x - y * floor(x / y); }
  inline float2 mod(float2 x, float2 y) { return x - y * floor(x / y); }
  // ...and so on for float3/float4
  ```
- This is worth fixing on general correctness grounds even if it isn't the cause of whatever
  specific bug prompted you to look at the translator — confirmed, in one real port's debugging
  session, to be a genuine latent bug independent of an unrelated shader-compiler reliability issue
  being investigated at the same time (don't assume finding and fixing this one bug means you've
  found the *whole* problem).

## Dead helper functions with unused `sampler2D` parameters can trip the shader compiler

Also relevant if your engine generates shader source programmatically (as libultraship/Fast3D-style
engines do) rather than hand-authoring each shader: it's easy to end up emitting a helper function
definition unconditionally into the generated source — e.g. a texture-fetch/filtering helper taking
a `sampler2D` parameter — while gating all of its *call sites* behind a runtime feature flag (e.g.
"is any texture actually bound for this draw"). When no texture is used, the function still gets
emitted but is never called, left as dead code with an unused opaque-resource-typed (`sampler2D`)
parameter. This was found and fixed in one real port as a plausible contributor to intermittent
Sony `SceShaccCg` runtime-compiler failures on specific no-texture shader variants — treat an
always-emitted helper function with an unused sampler parameter as a code smell worth eliminating
even before you've proven it's the root cause of a specific compiler failure.

**Mitigations:**
- Gate the helper function's *definition*, not just its call sites, behind the same condition that
  determines whether it's ever actually called — if nothing calls it, don't emit it.
- More generally, when shader source is generated programmatically from a set of feature flags,
  keep the emitted source's shape (which functions/variables exist at all) in sync with which
  features are actually active for that specific draw, rather than always emitting the "maximal"
  shader and relying on dead code being harmless — a runtime shader compiler is not guaranteed to
  treat truly dead code as inconsequential the way a mature, heavily-fuzzed desktop compiler would.

## Not verifying on real hardware

Broader than a vitaGL-specific pitfall, but worth repeating here because graphics code is exactly
the category most likely to "work" in an emulator while being subtly wrong on real hardware (see
[VitaSDK: debugging & tooling](../02-vitasdk/07-debugging-tooling.md)) — rendering correctness,
especially anything touching non-standard vitaGL extensions or memory-pool choices, deserves a real-
device check before being considered done.
