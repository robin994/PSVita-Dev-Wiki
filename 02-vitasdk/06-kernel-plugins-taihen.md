# Kernel Plugins & taiHEN

Most Vita homebrew — a game, a utility app, a downloader/manager — is a single ordinary user-mode
`eboot.bin` and never needs anything on this page. This page is about the minority of cases where a
project legitimately needs to reach *below* its own user-mode process: patching another process's
behavior, working around a retail-firmware restriction during a bootstrap sequence, or running a
background service independent of whether the main app is even open.

## What taiHEN provides

**taiHEN** (`tai_headers.h`/`taihen.h`) is the standard hooking/patching framework for Vita CFW.
Core primitives:

- **`taiHookFunctionImport`** — intercept a function a target module *imports* from another module,
  redirecting calls through your own function first (with a `tai_hook_ref_t` letting you call the
  original implementation from within your hook — the classic "wrap, optionally call through"
  pattern).
- **`taiHookFunctionExport`** — the export-side equivalent, hooking a function as seen by *callers*
  of the module that exports it.
- **`taiInjectData`** — patch raw bytes at a specific offset within a loaded module's memory image
  directly, rather than hooking a function boundary. Used for things a clean function hook can't
  reach — e.g., patching a hardcoded constant or a conditional check inline in existing code. This
  is inherently fragile: the target offset is specific to one exact build of the module being
  patched, so it silently breaks (or, worse, corrupts memory in a way that doesn't immediately
  crash) if the target firmware ships a different build of that module than the patch was written
  against.
- **`taiGetModuleInfo`** — resolve a loaded module's base address/metadata by name, a prerequisite
  for computing the right offset for `taiInjectData` or confirming a module is even loaded before
  hooking it.

## Kernel modules (`.skprx`) vs user-mode plugins (`.suprx`)

- A **kernel module** runs at the highest privilege level and can hook syscalls, patch other
  processes' memory, and touch hardware directly. Building one means linking against `-nostdlib`
  and kernel-specific stub libraries (`SceSysmemForDriver_stub`,
  `SceIofilemgrForDriver_stub`, and similar `ForDriver`/`ForKernel`-suffixed stub libraries — a
  different, more restricted API surface than ordinary user-mode code gets).
- A **user-mode plugin** runs inside another process's address space at ordinary user privilege —
  used to hook/patch the behavior of whatever process it's loaded into (commonly the running
  game/app itself, or the system shell), not to gain extra privilege.
- Both are declared to the build system with `vita_create_self(... CONFIG exports.yml)` — the YAML
  config declaring what the module exports for `taiHookFunctionImport`/`Export`-style consumers (or
  for the main app) to resolve against.
- The header split mirrors this privilege boundary directly: `vita-headers`' `include/` is divided
  into `psp2` (user-exported libraries — what an ordinary `.suprx`/main-app `eboot.bin` includes),
  `psp2kern` (kernel-exported libraries — what a `.skprx` includes), and `psp2common` (definitions
  shared by both). Including the wrong tree for what you're building either won't compile (missing
  kernel-only declarations in a user-mode target) or, worse, exposes APIs that don't actually exist
  at the privilege level you're building for.

## Common legitimate reasons a "normal app" ships a companion module

- **Bootstrapping around a retail-firmware check** that would otherwise block something homebrew
  legitimately needs to do early in a multi-stage install/setup flow (a debug-menu gate check, a
  filesystem redirect needed only during one specific setup step).
- **Relaunching the app at a precise point in a bootstrap sequence** — a user-mode plugin watching
  for a specific system event or file-removal signal and calling `sceAppMgrLaunchAppByName` at
  exactly the right moment, rather than the main app trying to detect and handle that condition
  itself from a privilege level that can't observe it.
- **Background daemons** — a separate, independently-scheduled process (not literally a
  kernel/user-mode plugin in the hooking sense, but often built and shipped alongside one) that
  polls for something (update checks, notifications) on a timer regardless of whether the main app
  is even running.

## Practical guidance

- Don't reach for kernel modules or taiHEN hooking by default — it's real added complexity,
  firmware-version fragility (especially anything using `taiInjectData` against a hardcoded offset),
  and a genuinely higher-risk category of bug (a bad kernel-level patch can crash or destabilize the
  whole system, not just your app's own process).
- If you do need one, keep the *title ID* and any other cross-referenced identifiers between the
  main app and its companion module(s) in sync manually — see
  [Homebrew app anatomy](03-homebrew-app-anatomy.md) — there's no compiler-enforced single source of
  truth linking separately-built binaries together.
- Treat any `taiInjectData`-based raw offset patch as inherently tied to one specific target module
  build; document clearly what build/firmware it was verified against, since it's exactly the kind
  of thing that silently stops working (or misbehaves) on a firmware update without an obvious error
  message pointing at the real cause.
