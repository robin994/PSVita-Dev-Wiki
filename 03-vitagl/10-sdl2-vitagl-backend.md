# SDL2 + vitaGL: the Northfear/SDL fork

Most desktop/mobile ports that use SDL2 for windowing expect `SDL_GL_CreateContext` /
`SDL_GL_SwapWindow` to hand off to a real OpenGL(-ES) implementation — which is exactly what code
built on Dear ImGui's `imgui_impl_sdl2.cpp` + `imgui_impl_opengl3.cpp` backends (see
[Dear ImGui: overview](../04-imgui/01-overview-core-concepts.md)) assumes it's talking to. VitaSDK's
**stock, bundled `libSDL2.a`** does not implement that path against vitaGL — its "vita" video driver
targets `sceGxm` directly for `SDL_Renderer`-based 2D rendering. If your project links stock SDL2 and
calls the generic `SDL_GL_*`/`ImGui_ImplOpenGL3_*` API expecting vitaGL underneath, you need a
different SDL2 build. **[Northfear/SDL](https://github.com/Northfear/SDL)** is the fork that adds
that: an SDL2 video driver backend that initializes and drives vitaGL itself.

This came up directly from Rinnegatamante (author of the original vitaGL-based libultraship Vita
port this wiki's porting section is modeled on) while reviewing a project that vendors stock Dear
ImGui + stock SDL2 for its GUI: using **local/stock Dear ImGui with SDL2 is the right call** (no
need for a Vita-specific imgui fork — see [imgui-vita backend](../04-imgui/02-imgui-vita-backend.md)
for when you *would* need one), but the SDL2 build underneath it specifically needs to be this fork's
`vitagl` branch, not VitaSDK's bundled one.

## Building it

```sh
git clone https://github.com/Northfear/SDL.git
cd SDL
git checkout vitagl
cmake -S. -Bbuild -DCMAKE_TOOLCHAIN_FILE=${VITASDK}/share/vita.toolchain.cmake \
      -DCMAKE_BUILD_TYPE=Release -DVIDEO_VITA_VGL=ON
cmake --build build -- -j$(nproc)
cmake --install build
```

`VIDEO_VITA_VGL=ON` is load-bearing — without it the fork builds the same stock sceGxm-direct video
driver VitaSDK ships, silently giving you back the thing you were trying to replace. The repo has
several other branches (`vita-2.0.3`, `vita-2.0.4`, `vita-2.0.9`, `vita-2.0.9-h3hd`,
`calloc_texture_vita`, `endscene_commandqueue`, `gxmfinish_remove`, `renderer-batch-draw-calls`) —
`vitagl` is the one with the vitaGL backend; the version-numbered branches track upstream SDL
releases without it, and the others are narrower experimental patches. Confirm you're actually on
`vitagl` before building; it's easy to end up on `main` and get the stock behavior back.

Related build-time flags this fork exposes, from its `docs/README-vita.md` (all default off):

| Flag | Effect |
|---|---|
| `VIDEO_VITA_VGL=ON` | Enables the GLES1/GLES2-via-vitaGL video driver — this page's whole subject. |
| `VIDEO_VITA_PVR=ON` | Enables GLES1/GLES2 (and, with `gl4es4vita` present, desktop GL 1.x/2.x) via PVR instead of vitaGL. Supports 720p/1080i via `SDL_setenv("VITA_RESOLUTION", "720"/"1080", 1)`; desktop-GL mode needs `SDL_setenv("VITA_PVR_OGL", "1", 1)` set before video-subsystem init. |
| `VIDEO_VITA_PIB=ON` | Enables GLES2 via PIB. |

None of these are mutually required with `VIDEO_VITA_VGL` — pick the one matching whichever backend
(vitaGL, PVR, PIB) your project actually links.

## What the vitaGL driver actually does (`src/video/vita/SDL_vitagles_vgl.c`)

This is the part worth understanding rather than treating as a black box, since it owns vitaGL's
entire lifecycle on your behalf:

- **`VITA_GLES_LoadLibrary`** calls `vglInitExtended(0, 960, 544, MEMORY_VITAGL_THRESHOLD, gxm_ms)`
  the first time it's invoked (`MEMORY_VITAGL_THRESHOLD` is a fixed 12 MiB in the fork; multisample
  mode is derived from `SDL_GL_MULTISAMPLESAMPLES` — 2 → `SCE_GXM_MULTISAMPLE_2X`, 4/8/16 → `_4X`,
  anything else → none) and never tears it down again for the process's lifetime — see
  [vitaGL: initialization & memory pools](02-initialization-memory-pools.md) for what those
  `vglInitExtended` parameters actually control. A module-static `vgl_initialized` flag guards this
  to exactly once; if your app also calls `vglInit`/`vglInitExtended` itself before creating the SDL
  window, you now have two initializers racing for the same library-global state — let the SDL driver
  own it instead.
- **`VITA_GLES_CreateContext`** doesn't create a real context object — vitaGL doesn't have a
  concept of multiple contexts. It force-sets `gl_config` to GLES 2.0 compatibility profile with
  8/8/8/8 color + 32-bit depth + 8-bit stencil, sets `SDL_WINDOW_FULLSCREEN`, and returns a pointer
  to the `vgl_initialized` flag as a dummy `SDL_GLContext` handle purely so SDL's context-tracking
  code has a non-null value to hold — nothing about that returned pointer is ever dereferenced as an
  actual context.
- **`VITA_GLES_MakeCurrent`** doesn't select between contexts (there's only ever the one, global
  vitaGL state) — it just does a `glFinish()` + full clear + another `glFinish()`, i.e. it behaves
  like a "reset the framebuffer" call each time you'd normally expect a context switch.
- **`VITA_GLES_SwapWindow`** is a thin wrapper over `vglSwapBuffers(GL_TRUE)`, with an
  `sceImeUpdate()` call folded in first if the on-screen keyboard (IME) is active — SDL's normal
  swap path is also where this fork pumps IME state, so if you bypass `SDL_GL_SwapWindow` (e.g.
  calling `vglSwapBuffers` directly yourself) you lose IME updates as a side effect, not just the
  swap.
- **`VITA_GLES_SetSwapInterval`** maps straight to `eglSwapInterval(0, interval)` — despite the
  `egl`-shaped name, this is vitaGL's own compatibility shim, not a real EGL display; see vitaGL's
  own source for what it does with the interval.

## Practical implications

- Because there's exactly one vitaGL global state and `LoadLibrary` initializes it once for the
  process, this driver is fundamentally single-window, single-context — consistent with vitaGL
  itself (see [vitaGL: common pitfalls](08-common-pitfalls.md)), not an added restriction from the
  SDL layer.
- `imgui_impl_opengl3.c`'s GL loader (or any other `SDL_GL_GetProcAddress`-based loader) will resolve
  symbols through `vglGetProcAddress`, not a real platform GL loader — so only symbols vitaGL
  actually implements resolve; anything outside vitaGL's supported subset returns null the same way
  it would calling `vglGetProcAddress` directly.
- If your project's window/GL init logs something like "Window OK" from stock SDL2 without ever
  actually exercising `SDL_GL_CreateContext`/`SDL_GL_SwapWindow` (e.g. window creation alone
  succeeds even on the stock, non-GL "vita" driver), that success does **not** confirm this fork
  is unnecessary — check specifically whether real frames make it to the screen once GL calls start
  flowing, not just whether window setup returned success.

## Sourcing note

Everything in this page comes from the fork's own `docs/README-vita.md` and reading
`src/video/vita/SDL_vitagles_vgl.c` / `SDL_vitavideo.c` directly (as of the `vitagl` branch, commit
history checked mid-2026) — not from separate documentation, since the fork doesn't have any beyond
that README. Community-sourced (Northfear's own work, credited in that README to prior contributions
from xerpi, cpasjuste, rsn8887, vitasdk/dolcesdk developers, and CBPS Discord members Graphene and
SonicMastr), not officially documented by Sony or SDL upstream.
