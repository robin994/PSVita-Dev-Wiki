# PVR_PSP2: a full PowerVR driver stack, as an alternative to vitaGL

**[PVR_PSP2](https://github.com/GrapheneCt/PVR_PSP2)** takes a fundamentally different architectural
approach from vitaGL to the same underlying problem — running OpenGL(-ES)-shaped code on Vita
hardware. Where vitaGL (see [Overview](01-overview.md)) implements a GL-like API surface *directly
on top of* Sony's native `sceGxm`, PVR_PSP2 instead implements a more complete **PowerVR driver
stack** — PVR referring to Imagination Technologies' PowerVR GPU architecture that Vita's GPU is
based on — providing lower-level hardware abstraction underneath multiple graphics standards at
once, rather than one GL-shaped wrapper over one lower-level API.

Per its own repository, the components are:

- **PVR2D** — a 2D graphics library.
- **WSEGL** — the Window System EGL layer.
- **IMGEGL** — an EGL implementation.
- **OpenGL ES 1.1 and 2.0** — full API ports, both marked complete upstream.
- A full Display Class API implementation, plus low-level code-generation utilities.

The recall of Northfear/SDL's `VIDEO_VITA_PVR` build flag (see
[SDL2 + vitaGL: the Northfear/SDL fork](10-sdl2-vitagl-backend.md)) is relevant here: that flag is
specifically what routes SDL2's GLES/desktop-GL rendering through **this** stack instead of vitaGL,
with resolution options (`SDL_setenv("VITA_RESOLUTION", "720"/"1080", 1)`) and a desktop-GL mode
(`SDL_setenv("VITA_PVR_OGL", "1", 1)`) vitaGL's own SDL backend doesn't expose the same way. If a
project needs GLES 1.1 specifically (vitaGL targets a GLES-*like* subset, not a strict 1.1
implementation) or desktop GL 1.x/2.x compatibility via `gl4es4vita`, PVR_PSP2 is the concrete
alternative to reach for rather than trying to force that shape of API onto vitaGL.

**Not independently evaluated for this wiki** — no on-device testing behind this page the way
vitaGL's own pages are backed by real findings (see
[Common pitfalls](08-common-pitfalls.md)'s emphasis on verifying any substitute graphics library
actually renders before trusting it). Treat "full stack, both ES versions complete" as the
project's own claim, not something confirmed here; apply the same "verify it actually renders on
real hardware before trusting the header/build success" discipline this wiki already applies to
vitaGL substitutes if evaluating this as an alternative.
