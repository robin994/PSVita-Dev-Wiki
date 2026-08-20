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

## vitaGL's own GLSL→Cg auto-translator (a third path)

Beyond "write raw Cg" and "precompile offline," some vitaGL builds also expose a **built-in
GLSL-to-Cg source translator** (`glsl_translator_process()` and friends, in vitaGL's own
`utils/glsl_utils.c`) that runs ahead of the runtime Cg compiler: you call `glShaderSource()`/
`glCompileShader()` with ordinary-looking GLSL text, vitaGL detects it's GLSL (not Cg) and rewrites
it into Cg source via a large preamble of macro `#define`s and inline overloaded helper functions
(`vglMul` standing in for GLSL's generic `*`/matrix-multiply overloading, `mod`/`fract`/`atan`
redirected to Cg-side equivalents, `texture2D` remapped, etc.) before handing the *result* to
`SceShaccCg`. This is how engines that were originally written against GLSL (e.g. libultraship's
Fast3D renderer, used by several N64-decompilation Vita ports) get to keep their GLSL shader source
mostly as-is rather than needing a hand-maintained Cg rewrite — but it means the Cg source that
actually reaches Sony's compiler is machine-generated and one layer removed from what you wrote,
which matters for the reliability issue below: a shader source problem could be either in your GLSL
or introduced by the translation step, and standalone-compiling *your* GLSL against a desktop GLSL
compiler doesn't rule out the latter.

## `SceShaccCg` runtime compilation can be genuinely unreliable, not just slow

Budgeting for "some compile-time cost" when choosing runtime shader compilation (see above) turns
out to understate the risk. **Confirmed on both real Vita hardware and the Vita3K emulator**: a
single `shark_compile_shader_extended()` call (vitaGL's wrapper around `SceShaccCg`, invoked from
`glCompileShader()`/`glLinkProgram()`) can take **10+ seconds of genuine wall-clock time** to either
succeed or fail, and — this is the part that breaks the usual debugging assumptions — **the exact
same Cg source, from the exact same build, has been observed to succeed on one run and fail with a
generic internal compiler error on another**, with no other change between runs. This was confirmed
non-source-related through multiple independent channels before accepting it as a genuine compiler
reliability issue rather than a bug in the shader:
- The shader in question compiled successfully standalone via `psp2cgc` (the offline Cg compiler),
  both independently by the vitaGL maintainer and separately by the project's own developer.
- Two real, unrelated bugs *were* found and fixed in the GLSL→Cg translation step along the way (a
  `mod()`/`fmod()` sign-semantics mismatch, and an unused helper function left with a dead
  `sampler2D` parameter — see [Common Pitfalls](08-common-pitfalls.md)) — fixing both was the right
  thing to do regardless, but neither one changed the intermittent-failure behavior, which is what
  finally separated "real translator bugs worth fixing" from "the compiler itself is just flaky for
  this input."

**What did *not* reliably fix it, tried in this order:**
- A one-time real-thread "pre-warm" compile of a trivial dummy shader before the shader in question
  is ever compiled (the same lazy-kernel-object pre-warm pattern described in
  [Kernel/Core APIs](../02-vitasdk/04-kernel-core-apis.md)) — helped in some runs, but the same
  failure still reproduced on other boot paths and other runs even with the pre-warm confirmed to
  have executed, ruling out "first-ever compile call" as the sole trigger.
- Retrying the failed compile call again immediately, or after a short `sceKernelDelayThread` — this
  made things *worse*: instead of failing (relatively) fast, the retried call would hang with no
  crash and no coredump. **A second `shark_compile_shader_extended()` call after a failed one is not
  a safe thing to do** — don't build a retry loop around this API assuming failure is a clean,
  recoverable state.

No fix for the underlying compiler reliability itself has been confirmed; the practical implication
for planning a project around runtime `SceShaccCg` compilation is that occasional multi-second
stalls and occasional hard failures on ordinary, valid, previously-working shaders should be treated
as an expected operating characteristic to design around (e.g. compile everything you'll need well
before it's needed and cache the result — see "Runtime vs offline shader compilation" above — rather
than something a specific code fix will make go away). A confirmed *workaround* that closes the
"genuine gap" identified below does exist, though — see "Proactive full-combo pre-warm" under Prior
art.

### A heap-pressure correlation, not just nondeterminism (confirmed, battleship-vita, 2026-08-20)

A real-hardware investigation on a fifth libultraship/Fast3D Vita port (the SSB64 port covered in
this wiki's own [case study](../07-porting-decompiled-games/02-case-study-ssb64.md)) found a
concrete correlation the "genuinely unreliable" framing above doesn't fully capture: a Fast3D shader
program compiled and linked successfully immediately after `InitWindow()`, while the same process's newlib heap usage was
still around 68 MiB — then, once game-resource loading had grown that usage to roughly 107–115 MiB,
*every subsequent, previously-untried* shader program failed with the same `fatal internal error line
-1`, on the same build, same device, same boot. This is consistent with `SceShaccCg` (or the
allocator path it depends on) becoming unreliable under heap pressure specifically, rather than
failing uniformly at random regardless of memory state — though it does not contradict the
"identical source, different outcome across runs" finding above, since two runs can easily differ in
how much heap has been consumed by the time a given shader is first requested. Treat "how much of the
process heap is already committed at compile time" as a variable worth controlling for, not just
logging, when chasing this failure mode.

### Prior art: how other libultraship/Fast3D Vita ports handle this

A survey of four shipped, real N64-decompilation Vita ports built on the same libultraship/Fast3D
renderer (Rinnegatamante's Ghostship/SM64, 2ship2harkinian/Majora's Mask, SpaghettiKart/Mario Kart
64, and Starship/Star Fox 64) turned up a consistent, but partial, answer:

- **None of them use vitaGL's `HAVE_SHADER_CACHE` build flag.** Instead, all four independently
  reimplement the *same* app-level pattern directly in their forked Fast3D `gfx_opengl.cpp`: on a
  shader-combo cache miss, compile+link as normal, then dump the linked program via
  `glGetProgramBinary()` to `ux0:data/<game>/shader_cache/<shader-id>_<magic>.bin`; on a hit, skip
  `glShaderSource`/`glCompileShader`/`glLinkProgram` entirely and load straight from
  `glProgramBinary()`. This is populated lazily, per-install, the first time each shader combo is
  actually hit during real gameplay — not built or bundled at compile time, and CI in all four repos
  builds Vita packages, if at all, without a shader-cache-priming step.
- **This only reduces the number of times you pay the runtime-compile risk (once per unique shader
  per install), it does not fix the underlying first-encounter reliability problem** — a shader
  combo your ROM produces that these ports never happen to exercise wouldn't be covered by their
  track record either, and none of them special-case dithering/noise shaders for Vita, so their
  clean history isn't direct evidence a specific noise-dither shape is safe.
- **No public writeup of this exact "fatal internal error on line -1" / nondeterministic-failure
  symptom was found** in vitaGL's issue tracker, any of the four ports' issue trackers, or general
  Vita homebrew community spaces — this appears to be either underreported or not previously
  isolated as cleanly as in this investigation.
- **The `vglSetupRuntimeShaderCompiler` lever: tried, ruled out (battleship-vita, 2026-08-20).**
  vitaGL exposes `vglSetupRuntimeShaderCompiler(shark_opt opt_level, int fastmath, int
  fastprecision, int fastint)` to override the optimizer settings passed into
  `shark_compile_shader_extended()` per-compile (vitaGL's own defaults are `SHARK_OPT_FAST`,
  `fastmath=1`, `fastint=1`, `fastprecision=0`). Two real-hardware experiments against the same
  reproducing shader both made things *worse*, not better: dropping `fastmath`/`fastint` to 0 (opt
  level left at default) turned the failure from its usual fast crash or ~13s success into an
  indefinite hang (4m45s+, no crash, no coredump); dropping the optimizer level all the way to
  `SHARK_OPT_SLOW` (O0, fastmath/fastint left at default) reproduced the exact same hang signature
  instead of fixing anything. **Any deviation from vitaGL's exact default optimizer settings turned
  this failure into a worse one (a silent hang) rather than resolving it** — treat both knobs as
  ruled out for this failure mode, not merely untested.
- **The proactive warm-priming gap: closed (battleship-vita, 2026-08-20).** None of the original four
  surveyed ports do any *proactive* warm-priming of their full shader-combo space (e.g. via a
  one-time "optimizing shaders" pass on first boot that walks every combo the ROM's draw calls can
  produce, extractable via a GBI trace — Starship's own docs describe exactly this trace-capture
  workflow) — they all compile strictly lazily, on first real encounter, mid-gameplay. A fifth port
  (SSB64, this wiki's own [case study](../07-porting-decompiled-games/02-case-study-ssb64.md)) closed
  this gap directly: it precompiled the 41 unique shader pairs observed across full hardware
  boot-and-attract-mode captures immediately after `InitWindow()` — while the newlib heap was still
  small, per the heap-pressure correlation above — and kept every successful program in the
  renderer's normal in-memory cache (not the unsafe on-disk `glProgramBinary` cache). Confirmed on
  hardware: 41/41 pre-warmed programs succeeded, and no further `shader link failed` occurred for
  the rest of that session. This doesn't fix `SceShaccCg`'s underlying reliability problem — a combo
  outside the captured set would still risk hitting it lazily, under heap pressure, exactly as
  before — but it demonstrates the proactive-priming approach genuinely works when the combo space is
  known and primed early enough.

## Practical guidance

- If you don't have a specific shading requirement the default pipeline can't satisfy, don't
  introduce custom shader complexity — the default path handles a surprising amount of ordinary
  UI/2D rendering cleanly.
- If porting shader code from another platform, budget real time for a Cg rewrite, not a
  find-and-replace — GLSL and Cg diverge meaningfully in syntax and built-in function/semantic
  naming despite the surface-level family resemblance.
- Decide deliberately between runtime (`vitashark`) and offline shader compilation based on your
  actual iteration-speed vs startup-cost tradeoff, rather than defaulting to whichever a starter
  project template happens to use — and budget for the *reliability* cost of runtime compilation
  above, not just the raw speed cost.
- If using vitaGL's GLSL auto-translator, know it's there and that it's a real, separate
  transformation step — when debugging a shader-compile failure, verifying your original GLSL
  compiles fine elsewhere doesn't rule out the machine-generated Cg it gets turned into.
