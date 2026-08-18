# VitaSDK

**VitaSDK** (`vitasdk.org`) is the open-source, community-maintained toolchain and header/library
collection used to build homebrew for the PS Vita. It's not Sony's official SDK (which was never
publicly released) — it's a from-scratch, community-reverse-engineered reimplementation of the
headers, import-stub libraries, and build tooling needed to produce code that runs against the
Vita's real system libraries.

Practically, VitaSDK gives you: a cross-compiler targeting `arm-vita-eabi` (a GCC toolchain), header
files for the system API surface (`psp2/*.h`), stub libraries that let you link against system
functions without having their real implementation available at build time (the real
implementation lives in the console's firmware), and CMake integration (`vita.toolchain.cmake`,
`vita.cmake`) that knows how to produce the Vita's actual executable/package formats.

## Pages in this section

1. [Overview & toolchain](01-overview-toolchain.md) — what VitaSDK actually is, install layout, the cross-compiler
2. [Project setup & build system](02-project-setup-build-system.md) — CMake integration, common pitfalls, linking gotchas
3. [Homebrew app anatomy](03-homebrew-app-anatomy.md) — eboot.bin, param.sfo, VPK structure, title IDs
4. [Kernel/core APIs](04-kernel-core-apis.md) — threading, memory, file I/O, module loading
5. [System libraries](05-system-libraries.md) — display, input, audio, dialogs, app management
6. [Kernel plugins & taiHEN](06-kernel-plugins-taihen.md) — when and how homebrew reaches below user mode
7. [Debugging & tooling](07-debugging-tooling.md) — VitaShell, real-time logging, `.psp2dmp` crash dump analysis, emulation vs real hardware
8. [Best practices](08-best-practices.md) — a consolidated checklist for writing solid VitaSDK code
9. [FMOD audio (FMOD-PSV)](09-fmod-audio.md) — the community FMOD Ex/Designer bridge, relevant when
   porting a game/engine (e.g. an Android port) whose audio layer already targets FMOD
