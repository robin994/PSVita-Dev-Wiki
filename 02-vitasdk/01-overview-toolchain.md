# VitaSDK: Overview & Toolchain

## What VitaSDK actually is

Sony never publicly released an official Vita SDK. Everything the homebrew community builds against
— the system call surface, header definitions, function signatures, even in many cases the
numeric NIDs (function identifiers the firmware actually resolves imports by, since the Vita's
dynamic linking is NID-based rather than symbol-name-based like a conventional ELF loader) — comes
from years of community reverse engineering, consolidated into the `vitasdk` GitHub organization's
projects: `vita-headers` (the header/NID database), `vita-toolchain` (tools to produce Vita-format
executables from a standard cross-compiler's output), `vdpm` (a package manager for prebuilt
third-party library ports), and the SDK installer/build scripts that tie it together. The generated
API reference for `vita-headers` (doxygen, rebuilt on every push to its master branch) is hosted at
http://vitasdk.github.io/vita-headers — the fastest way to look up a function/struct's actual
declaration without grepping the installed SDK by hand.

Because of this origin, **the header surface is not 100% complete or 100% guaranteed-accurate** —
some functions are documented from partial reverse engineering, some struct layouts are inferred
from observed behavior rather than an authoritative source, and occasionally a function you need
isn't in the shipped headers at all and has to be hand-declared with a manually-supplied signature
(common for less-used or newer firmware-introduced APIs). This isn't a criticism of the project —
it's a genuinely impressive, actively maintained community effort — but it's important context:
when something behaves unexpectedly, "the header/assumed behavior might just be wrong or
incomplete" is a real, non-rare possibility, not just a place to look last.

## The cross-compiler

VitaSDK ships (or builds) a GCC-based cross-compiler targeting the triple `arm-vita-eabi` —
producing ARMv7-A/Thumb-2 object code appropriate for the Cortex-A9 cores (see
[Hardware: CPU architecture](../01-hardware/02-cpu-architecture.md)). It's invoked as
`arm-vita-eabi-gcc`/`arm-vita-eabi-g++` and friends, same as any other GNU cross-toolchain — nothing
exotic about the compiler invocation itself once your build system is pointed at it correctly.

## Install layout

A typical install (commonly at `/usr/local/vitasdk` on Linux/macOS, referenced via the `$VITASDK`
environment variable most build setups expect) contains:

- `arm-vita-eabi/include/` — the C/C++ standard library headers (newlib-based libc) plus
  `psp2/*.h`, the Vita system API headers (`kernel.h`, `display.h`, `ctrl.h`, `avplayer.h`, and so
  on — one header per system library, broadly).
- `arm-vita-eabi/lib/` — static libraries, including the **`*_stub`** import libraries
  (`SceCtrl_stub`, `SceDisplay_stub`, `SceKernelThreadMgr_stub`, ...) that satisfy the linker for
  calls into system functions whose real implementation is provided by the console's firmware at
  runtime, not by anything in your compiled binary.
- `share/vita.toolchain.cmake` / `share/vita.cmake` — CMake integration providing the toolchain file
  and the Vita-specific build macros (`vita_create_self`, `vita_create_vpk`, etc.) — see
  [Project setup & build system](02-project-setup-build-system.md).
- `bin/` — the toolchain binaries themselves, plus VitaSDK-specific utilities like `vita-elf-create`,
  `vita-make-fself`, `vita-mksfoex` (the tools that convert a standard ELF into the Vita's actual
  executable formats — see [Homebrew app anatomy](03-homebrew-app-anatomy.md)).

## Stub libraries and NIDs

Every system function a homebrew app calls is resolved at load time by **NID** (a 32-bit numeric
identifier derived from the function's real name via a known hash), not by conventional symbol name
resolution. The `*_stub` libraries you link against don't contain real function bodies — they
contain just enough metadata for the linker to produce the correct import table entries, which the
firmware's module loader then resolves against the actual running system library by NID at process
launch. This is invisible to you as an application author writing normal C/C++ calls — you just
call `sceCtrlPeekBufferPositive(...)` like any other function — but it explains why linking against
the *wrong* stub library version, or a header/NID mismatch between your VitaSDK version and the
target firmware, can produce mysterious runtime failures (unresolved import, or worse, a resolved-
but-wrong function) that a normal desktop toolchain mismatch wouldn't manifest the same way.

## Versioning and update cadence

VitaSDK is actively maintained but doesn't follow a strict release-versioning scheme the way, say, a
numbered SDK release from a vendor would — you generally track it via git/package-manager updates
rather than "SDK version 3.x vs 4.x." Because of the reverse-engineering origin, occasional breaking
changes do land (a struct layout correction, a function signature fix) as better information becomes
available — worth being aware of if you maintain code across a long timespan and something that
compiled cleanly a year ago suddenly doesn't after an SDK update; check whether an upstream
correction is the cause before assuming your own code regressed. `vita-headers` actually flags this
class of change to consumers deliberately, via `vita.header_warn.cmake` — a CMake hook that emits a
build-time notice when the headers you're building against introduce a backwards-incompatible
change since the last version you might have pinned. The NID database itself (`db/`) is organized
per-firmware-version (separate directories per SDK/firmware release the community has reverse
engineered), and `PSP2_SDK_VERSION` (`psp2common/defs.h`) is available at compile time if you need
to conditionally guard code against a specific header-surface era — relevant since this project's
own PSM-runtime bootstrap sequence is exactly the kind of code that's sensitive to
firmware/SDK-version-specific behavior.
