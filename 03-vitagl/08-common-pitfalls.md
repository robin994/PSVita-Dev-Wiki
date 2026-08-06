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

## Not verifying on real hardware

Broader than a vitaGL-specific pitfall, but worth repeating here because graphics code is exactly
the category most likely to "work" in an emulator while being subtly wrong on real hardware (see
[VitaSDK: debugging & tooling](../02-vitasdk/07-debugging-tooling.md)) — rendering correctness,
especially anything touching non-standard vitaGL extensions or memory-pool choices, deserves a real-
device check before being considered done.
