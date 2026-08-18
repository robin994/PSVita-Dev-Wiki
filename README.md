# PS Vita Development Wiki

A reference wiki for PlayStation Vita homebrew development: the hardware itself, VitaSDK (the
toolchain and system libraries), vitaGL (the OpenGL-subset-over-sceGxm graphics library), Dear
ImGui as ported for the Vita (imgui-vita), vita2d (a high-level 2D graphics library, the
no-porting-required alternative to vitaGL for new 2D apps), and the general methodology for porting
an OpenGL-based library to vitaGL when that *is* what's needed. Written to stand on its own — not
tied to any one project — so it can be used as a general-purpose reference whenever any of these
come up again.

Each section moves from macro concepts (what the thing is, how it fits into the whole picture) down
to micro topics (specific APIs, specific gotchas, specific code patterns), and closes with a
best-practices summary.

## Purpose: written for AI-assisted development

This wiki exists specifically to be fed to an AI coding assistant as context for PS Vita homebrew
work — the goal is to give a model enough grounded, verified knowledge to answer questions about
Vita development correctly and usefully, instead of guessing at APIs, hallucinating plausible-
sounding but wrong hardware behavior, or repeating outdated/community-folklore claims as fact. Every
page is written with that use case in mind: precise terminology, explicit callouts for what's
officially documented versus community-reverse-engineered convention (see "Scope and honesty about
sourcing" below), and concrete, real findings (often from actual on-device testing) rather than
generic advice that sounds right but hasn't been verified.

This is a living document, not a one-time snapshot: new pages and corrections get added as new
findings come up in real projects — a bug root-caused, a gotcha hit and worked around, a piece of
undocumented behavior confirmed on real hardware. Treat anything here as the best current
understanding, kept up to date deliberately, not a finished/frozen reference.

## Sections

1. **[PS Vita Hardware](01-hardware/README.md)** — the SoC, GPU, memory hierarchy, storage,
   input devices, multimedia decoders, networking, and the system software layers everything else
   in this wiki runs on top of.
2. **[VitaSDK](02-vitasdk/README.md)** — the open-source toolchain and headers used to build
   homebrew: project setup, the kernel/user API surface, plugins, debugging, and conventions.
3. **[vitaGL](03-vitagl/README.md)** — a GLES-like API implemented on top of Sony's native
   `sceGxm`, letting you write mostly-portable OpenGL code that actually runs on the Vita's GPU.
4. **[Dear ImGui / imgui-vita](04-imgui/README.md)** — the immediate-mode GUI library and its
   Vita-specific backend (`imgui_impl_vitagl`), for building tool-like or menu-driven UIs quickly.
5. **[vita2d](05-vita2d/README.md)** — a high-level 2D graphics library built directly on `sceGxm`,
   not GL-shaped at all; the right starting point for a *new* 2D app when you're not porting
   existing OpenGL code. Includes a full worked case study of VHBB, a real app-store client built
   entirely on it.
6. **[Porting OpenGL libraries to vitaGL](06-porting-opengl-libraries-to-vitagl/README.md)** — the
   general methodology (extracted from imgui-vita's real, verified port) for when you *do* need to
   bring an existing OpenGL-based library onto vitaGL, plus a planning-stage case study applying it
   to RmlUi.
7. **[Porting decompiled console games](07-porting-decompiled-games/README.md)** — a different
   problem from section 6: bringing a whole decompiled game (not a single library) to the Vita,
   using the libultraship/Shipwright pattern behind community ports like Ghostship and
   SpaghettiKart. Includes a case study applying it to a Super Smash Bros. 64 port, with real
   findings from actual real-hardware bring-up.
8. **[Porting Android apps/games](08-porting-android-apps/README.md)** — a third porting shape:
   running an Android APK's compiled native library directly on Vita (no source needed), via a
   `.so` loader/relocator plus a fake JNI/JVM surface.

## How to read this

If you're starting from zero, read in order: hardware gives you the mental model everything else is
built on, VitaSDK is the ground floor of "how do I even get code running on the device," and then
the fork depends on what you're building. Writing a new 2D app from scratch and not porting
anything? Read vita2d next — it's the gentler, no-GL-knowledge-required path, proven by the VHBB
case study. Bringing an existing OpenGL-based library (a UI toolkit, an engine) onto the Vita
instead? Read vitaGL, then imgui-vita as a real worked example, then the porting-methodology
section for how to generalize that to a different library. Porting a whole decompiled game (a
libultraship/Shipwright-family project, or similar) rather than a single library? Read vitaGL for
the graphics foundation, then go straight to section 7 — it's a different shape of problem from
section 6 and has its own methodology. Porting an Android game/app instead, with no source and only
a compiled `.so`? Section 8 is its own methodology again, closer in spirit to "run someone else's
binary" than to either porting shape above. If you already know roughly what you're
doing, jump straight to the section/page you need — each page is written to be self-contained
enough to skim on its own.

## Scope and honesty about sourcing

This wiki reflects a mix of official VitaSDK documentation (`docs.vitasdk.org`), the vitaSDK and
vitaGL/imgui-vita source trees themselves, and accumulated homebrew-community knowledge and
convention. The Vita's official SDK and most low-level hardware documentation were never publicly
released by Sony — a lot of what the homebrew scene knows comes from reverse engineering
(`vitasdk/vita-headers`, psdevwiki-era research, and projects like `vita-toolchain`), so where
something is inferred/community-convention rather than officially documented, that's called out
explicitly rather than stated as fact.
