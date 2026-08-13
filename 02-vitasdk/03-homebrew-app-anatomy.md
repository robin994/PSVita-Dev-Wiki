# Homebrew App Anatomy

## The pieces of a VPK

A `.vpk` is the installable package format homebrew installers (VitaShell foremost among them)
consume. Unzip one and you'll typically find:

- **`eboot.bin`** — the main executable, a SELF (signed ELF) — see below. This is what actually
  runs when the app launches.
- **`sce_sys/param.sfo`** — metadata: title ID, app version, app name, and various flags. Generated
  by `vita-mksfoex`/the `vita_create_vpk` CMake macro from values you supply (`TITLEID`, `VERSION`,
  `NAME`), or extra flags via `VITA_MKSFOEX_FLAGS` — most notably `ATTRIBUTE2`, which raises the
  app's default 256 MiB RAM budget up to the full 365 MiB partition; see
  [Hardware: memory architecture](../01-hardware/04-memory-architecture.md#your-apps-actual-ram-budget-the-attribute2-paramsfo-flag).
- **`sce_sys/icon0.png`**, **`sce_sys/livearea/contents/*.png`+`template.xml`** — the LiveArea tile
  assets (icon, background, layout) shown on the system's home screen for the installed app.
- Any bundled data files the app needs at runtime (fonts, additional icons, bundled default assets)
  — placed at whatever in-package path your `vita_create_vpk` `FILE` arguments specify, and readable
  at runtime via the `app0:` mount point (see
  [Hardware: storage & filesystem](../01-hardware/05-storage-filesystem.md)).
- Any companion **`.suprx`/`.skprx`** modules the app depends on (a kernel driver, a user-mode
  plugin hooked into the main process) — see
  [Kernel plugins & taiHEN](06-kernel-plugins-taihen.md).

## SELF, ELF, and why the distinction matters

Your compiler produces a normal ELF executable first. VitaSDK's tooling (`vita-elf-create` +
`vita-make-fself`, wrapped by the `vita_create_self` CMake macro) then converts that into a **SELF**
(Signed ELF) — the format the console's loader actually expects, with Sony's own signing/encryption
header wrapped around the payload on retail hardware. Homebrew SELFs are built with the `UNSAFE`
flag, producing a SELF that skips real cryptographic signing in a way CFW's patched loader accepts
— retail, unmodified firmware would refuse to run it. This is *why* Vita homebrew inherently
requires a modified console (CFW) to run at all, unlike, say, a sideloaded app on a platform that
merely requires enabling a developer-mode toggle.

## Title IDs

A **title ID** is a fixed-format identifier (commonly 9 characters, e.g. `ABCD12345`-shaped)
uniquely identifying an installed app in the system's app database. It's set at build time
(`VITA_TITLEID` in a typical `CMakeLists.txt`) and shows up in several places you need to keep in
sync if you ever change it:

- The install path the app's own package ends up at (`ux0:app/<TITLEID>/`).
- Any companion kernel/user-mode plugin that needs to relaunch or otherwise reference the app by
  title ID (`sceAppMgrLaunchAppByName` and similar calls take a title ID, not a friendly name).
- Any hardcoded path literal elsewhere in your own source that assumes a specific `ux0:app/<TITLEID>`
  location rather than deriving it — a common source of "I renamed my title ID and now half the app
  is broken in ways that don't show up as build errors" bugs, since these are runtime string
  literals, not something the compiler can catch for you.

**Practical rule**: if a project has more than one place referencing its own title ID (a build-time
CMake variable, a companion plugin's C source, a handful of hardcoded path strings), treat those as
one fact that must be kept in sync manually — there's no single source of truth the toolchain
enforces for you across separately-compiled binaries.

## Self-update flows

Homebrew apps that want to self-update in-app (download a newer VPK/PSARC, extract, relaunch)
typically use **`scePromoterUtil`** — the same mechanism the system's own installer uses to register
an extracted package as a properly-installed app — extracting the update payload over the app's own
`ux0:app/<TITLEID>` directory and re-promoting it, rather than trying to replace the running
`eboot.bin` file directly while it's executing (which the filesystem/loader won't cleanly allow
anyway). Getting the extraction target path and title ID consistent (see above) is the most common
point of failure in a self-update feature.
