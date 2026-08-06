# System Software Layers

## Privilege layers

The Vita's system software is layered by privilege, and understanding the layering matters for
knowing what a piece of homebrew code is actually allowed to do without extra help:

- **Kernel mode** — the highest privilege level. Kernel modules (`.skprx` files) can hook arbitrary
  syscalls, patch running code in other processes, and touch hardware directly without going
  through any user-mode API surface. Only reachable at all on a modded ("custom firmware"/CFW)
  console — retail firmware doesn't let arbitrary code run at kernel privilege.
- **User mode** — where ordinary applications (including homebrew `eboot.bin`s) run. User-mode code
  talks to the kernel exclusively through defined syscall stubs (the `*_stub` libraries you link
  against — see [VitaSDK](../02-vitasdk/README.md)). A normal homebrew app never needs kernel
  access for typical functionality (graphics, input, storage, networking, media are all reachable
  through user-mode syscalls).
- **User-mode plugins (`.suprx`)** — code that runs *inside* another process's user-mode address
  space (commonly the running game/app, or the system shell process) rather than as its own
  standalone process — used for hooking/patching behavior of other software, not for building a
  standalone app.

Most homebrew — including typical GUI applications, games, downloader/manager tools — is a plain
user-mode `eboot.bin` and needs none of the kernel/plugin machinery. Kernel modules and user-mode
plugins show up specifically when a project needs to hook into or modify behavior *outside* its own
process (system-level patches, bootstrap workarounds for CFW-specific quirks, background daemons
that need capabilities a normal app process doesn't have).

## taiHEN

**taiHEN** is the de facto standard hooking/patching framework for Vita CFW — it provides the
primitives (`taiHookFunctionImport`, `taiHookFunctionExport`, `taiInjectData`, and friends) that
kernel modules and user-mode plugins use to intercept function calls or patch code/data in a running
process, without each project reinventing its own hooking mechanism. If you see `#include
<taihen.h>` and calls like `taiHookFunctionImport(...)` in Vita homebrew source, that's a taiHEN
consumer — most commonly a kernel-mode `.skprx` or user-mode `.suprx` plugin, not a standalone app
(standalone apps generally have no reason to hook anything, their own or otherwise).

## SceShell and the boot chain

`SceShell` is Sony's own system UI process (the LiveArea home screen, app launcher, system
settings). Installed homebrew apps get their own LiveArea tile and launch like any other app once
"promoted" into the system's app database via `scePromoterUtil` — the mechanism used by both the
official installer and by homebrew VPK installers (like VitaShell) to register a package as a
proper launchable app rather than a loose file.

**taiHEN-based CFW setups** (`henkaku`/`h-encore`-derived and successors) typically load a
persistent kernel-level taiHEN instance early in the boot chain (often via a config that
auto-launches on boot, commonly named something like `PSVSHELL`/`ur0:tai/config.txt` in various
CFW generations), which is what makes kernel-level hooks and non-retail plugin loading possible for
everything that boots afterward — including homebrew installed as ordinary apps that may themselves
rely on a companion kernel plugin for functionality retail firmware wouldn't otherwise permit
(bootstrap workarounds, FIOS overlay redirection tricks, etc.).

## VitaShell

**VitaShell** is the de facto standard homebrew file manager / VPK installer / FTP server for the
platform — most homebrew distribution assumes the end user has it installed, since it's the primary
way non-technical users get `.vpk` packages onto and installed on a modded console, and its
built-in FTP server is the most common way developers transfer freshly built packages/test assets to
a real device during development (see
[VitaSDK: debugging & tooling](../02-vitasdk/07-debugging-tooling.md)).

## Practical takeaways

- A typical GUI/tool/game homebrew app only needs user-mode syscalls — don't reach for kernel
  modules or taiHEN hooking unless you have a concrete need to affect something outside your own
  process.
- If a project *does* ship a kernel module or user-mode plugin alongside its main app, that's
  usually there to solve a specific bootstrap/compatibility problem (patching a system check,
  redirecting a filesystem path, relaunching the app at a specific point in a multi-stage install
  flow) — not a sign that kernel access is generally required for that category of app.
- Assume your end users have VitaShell (or an equivalent) installed for distribution/testing
  purposes; it's the de facto standard, not an optional nice-to-have.
