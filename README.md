# PS Vita Development Wiki

A reference wiki for PlayStation Vita homebrew development: the hardware itself, VitaSDK (the
toolchain and system libraries), vitaGL (the OpenGL-subset-over-sceGxm graphics library), and
Dear ImGui as ported for the Vita (imgui-vita). Written to stand on its own — not tied to any one
project — so it can be used as a general-purpose reference whenever any of these come up again.

Each section moves from macro concepts (what the thing is, how it fits into the whole picture) down
to micro topics (specific APIs, specific gotchas, specific code patterns), and closes with a
best-practices summary.

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

## How to read this

If you're starting from zero, read in order: hardware gives you the mental model everything else is
built on, VitaSDK is the ground floor of "how do I even get code running on the device," vitaGL is
the graphics layer most homebrew ends up using, and imgui-vita is a common choice for the UI layer
on top of that. If you already know roughly what you're doing, jump straight to the section/page you
need — each page is written to be self-contained enough to skim on its own.

## Scope and honesty about sourcing

This wiki reflects a mix of official VitaSDK documentation (`docs.vitasdk.org`), the vitaSDK and
vitaGL/imgui-vita source trees themselves, and accumulated homebrew-community knowledge and
convention. The Vita's official SDK and most low-level hardware documentation were never publicly
released by Sony — a lot of what the homebrew scene knows comes from reverse engineering
(`vitasdk/vita-headers`, psdevwiki-era research, and projects like `vita-toolchain`), so where
something is inferred/community-convention rather than officially documented, that's called out
explicitly rather than stated as fact.
