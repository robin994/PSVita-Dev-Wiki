# Case Study: Planning a Port of RmlUi to vitaGL

**Status: unexecuted planning guidance, not a description of working code.** RmlUi
(`TapeRTS/libRocket-RmlUi`, "the HTML/CSS User Interface library evolved," a maintained continuation
of the old libRocket project) has no known existing Vita/vitaGL port. This page applies the
[general methodology](01-methodology.md) to it as a worked planning example — every claim here is
either read directly from RmlUi's own repository structure/headers, or is explicit forward-looking
guidance, never presented as verified working behavior.

## The three interfaces to implement

Confirmed by inspecting the repo: RmlUi's `Backends/` folder cleanly separates three concerns that
are easy to conflate when skimming its examples:

| Interface | Lives in | Role |
|---|---|---|
| `Rml::RenderInterface` | `RmlUi_Renderer_GL2.cpp`/`_GL3.cpp` (reference implementations) | Converts RmlUi's already-triangulated geometry + textures into real draw calls — **this is the actual porting work** |
| `Rml::SystemInterface` | `RmlUi_Platform_*.cpp` | Clock/time source, logging |
| `Rml::FileInterface` | (app-provided) | Opens `.rml`/`.rcss`/font/image files from disk |

A fourth category, `RmlUi_Backend_*.cpp` (e.g. `RmlUi_Backend_SDL_GL2.cpp`), is just glue code
combining a specific windowing library with a specific renderer for RmlUi's own desktop samples —
**not** part of the library's actual interface contract, and not something a Vita port needs to
resemble at all.

## Applying methodology principle #2: which reference renderer to study

RmlUi ships both `RmlUi_Renderer_GL2.cpp` (fixed-function-style) and `RmlUi_Renderer_GL3.cpp`
(core-profile: VAOs, uniform buffers). Per
[the general methodology](01-methodology.md#2-start-from-the-librarys-simplest-existing-gl-backend-as-a-map)
— and directly consistent with what [imgui-vita's real port](01-methodology.md#2-start-from-the-librarys-simplest-existing-gl-backend-as-a-map)
demonstrates about which GL era vitaGL implements well — **`RmlUi_Renderer_GL2.cpp` is the right
study target, not `_GL3.cpp`.**

## Where this port would likely need principle #3 (reformulation)

RmlUi's `RenderInterface::CompileGeometry` model assumes the renderer can build and keep a reusable
GPU-side geometry handle. Whether that maps directly onto vitaGL's mapped-pointer draw path, or
needs the same kind of CPU-side reformulation imgui-vita applied to indexed draws (see
[methodology principle #3](01-methodology.md#3-dont-force-an-operation-the-target-api-doesnt-offer-comfortably--reformulate-it)),
is exactly the kind of thing that can only be determined by actually attempting milestone M2 below
— flagged here as a known open question, not a solved one.

## A milestone roadmap (verify-before-proceeding, not a schedule)

1. **M0 — Core library builds for VitaSDK.** `libRmlUi.a` cross-compiles with Lua/SVG/Debugger/
   Samples disabled. Verify: a trivial program including `<RmlUi/Core.h>` links.
2. **M1 — Minimal `SystemInterface`+`FileInterface`.** `Rml::Initialise()` succeeds. Verify: no
   crash on init/shutdown.
3. **M2 — `RenderInterface`: one hardcoded colored triangle.** The real test of principle #2/#3
   above. Verify: visible on **real hardware**, not just an emulator (see
   [vitaGL: common pitfalls, "Not verifying on real hardware"](../03-vitagl/08-common-pitfalls.md)).
4. **M3 — Textures.** Verify: correct orientation/channel order (a classic port bug — see
   [vita2d's own RGBA byte-order pitfall](../05-vita2d/03-primitives-and-drawing.md) for the same
   class of mistake in a different library).
5. **M4 — Scissor/clipping**, load-bearing for RmlUi's `overflow` handling before anything else
   feels "correct." Directly comparable to
   [vita2d's region clipping](../05-vita2d/06-render-targets-and-clipping.md) — same concept, this
   library's own name for it.
6. **M5 — Fonts via FreeType** (already available prebuilt in VitaSDK).
7. **M6 — A real `.rml` document with touch/analog input.**
8. **M7 — Batching/perf pass** on real hardware with a non-trivial document.

## Known risk areas, stated honestly

- Memory: RmlUi's glyph atlases + image textures accumulate quickly against the Vita's limited RAM;
  plan explicit texture release (`RenderInterface::ReleaseTexture`) from M3 onward, not as an
  afterthought.
- This is unverified against real vitaGL shader/VAO support depth — M2 is where that gets tested for
  real, and the outcome could require deviating from this plan.

## Practical takeaway

Read this page as "here's how you'd start, and here's exactly where to stop and verify before
assuming the next step will just work" — not as a guarantee the port is straightforward. The
[VHBB case study](../05-vita2d/09-case-study-vhbb.md) remains the lower-risk alternative for anyone
whose actual goal is "ship a Vita app," not specifically "get RmlUi running on Vita."
