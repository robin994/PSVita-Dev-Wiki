# vitaGL: Shaders

## Two paths: fixed-function-style default, or custom Cg shaders

vitaGL provides a built-in default shader pipeline covering common fixed-function-era needs
(textured/untextured, vertex-colored geometry) so that a lot of straightforward 2D/UI rendering
never needs to touch shader code at all — you just set GL state (texture bound or not, blend mode,
color) and draw, the way you would against classic fixed-function OpenGL ES 1.x. For anything
needing custom shading beyond that default path, vitaGL supports loading real custom vertex/fragment
shaders.

## The shader toolchain: Cg, not GLSL

Sony's native shader language for sceGxm is **Cg** (Nvidia's older "C for Graphics" shading
language, which Sony adopted for the PlayStation shader toolchains of this era), compiled via Sony's
own Cg-based compiler tooling — `SceShaccCg` at the system-library level, with **`vitashark`** being
the community project providing a *runtime* Cg-to-GXP shader compiler usable from within a running
homebrew app (as opposed to only being able to precompile shaders offline ahead of time). This means
custom vitaGL shaders are written in Cg syntax, not GLSL — a real, non-trivial porting consideration
if you're bringing over shader code originally written against desktop/mobile OpenGL, since Cg and
GLSL, while related in spirit, are not source-compatible.

## Runtime vs offline shader compilation

- **Runtime compilation** (via `vitashark`) lets an app ship Cg *source* and compile shaders on the
  device at load time — more flexible (easier iteration, shaders can be modified without a full
  rebuild pipeline), at the cost of doing real compilation work on relatively modest hardware every
  time the app starts or a new shader is first needed.
- **Offline/precompiled** shaders (compiled ahead of time on a dev machine into Sony's native GXP
  bytecode format) avoid that runtime cost entirely, at the cost of needing a proper build-time
  shader compilation step in your toolchain and losing the "edit and immediately retest on-device"
  convenience.

Most straightforward homebrew UI/2D work never needs custom shaders at all and stays entirely within
vitaGL's default fixed-function-style path — reach for custom Cg shaders when you have a genuine need
(custom lighting/post-processing effects, a specific visual technique the default pipeline can't
express), not as a default starting point.

## Practical guidance

- If you don't have a specific shading requirement the default pipeline can't satisfy, don't
  introduce custom shader complexity — the default path handles a surprising amount of ordinary
  UI/2D rendering cleanly.
- If porting shader code from another platform, budget real time for a Cg rewrite, not a
  find-and-replace — GLSL and Cg diverge meaningfully in syntax and built-in function/semantic
  naming despite the surface-level family resemblance.
- Decide deliberately between runtime (`vitashark`) and offline shader compilation based on your
  actual iteration-speed vs startup-cost tradeoff, rather than defaulting to whichever a starter
  project template happens to use.
