# System UI integration libraries

Small, focused libraries that hook a homebrew app or plugin into a piece of the Vita's own system
UI, rather than building an equivalent from scratch in vitaGL/vita2d/imgui-vita. Reach for these
when the goal is looking/behaving consistent with the system rather than a fully custom UI.

## QuickMenuReborn — custom widgets in the system Quick Menu

**[QuickMenuReborn](https://github.com/Ibrahim778/QuickMenuReborn)** lets a plugin add its own
widget to the Vita's Quick Menu (the system-level overlay, distinct from any in-app menu) without
hand-rolling the low-level Quick Menu integration yourself. Integration is deliberately
config-file-free: a compiled plugin dropped into QuickMenuReborn's own folder is picked up
automatically, unlike a normal taiHEN plugin's `config.txt` entry (see
[Kernel plugins & taiHEN](06-kernel-plugins-taihen.md)). Supports both VitaSDK/DolceSDK and VDSuite
(see [Alternative toolchains & deployment](11-alternative-toolchains-deployment.md#vdsuite--a-third-sdktoolchain-lineage))
as build targets, and ships a sample plugin covering most of its capability surface.

## libAppSettings — the native settings-dialog bridge

**[libAppSettings](https://github.com/GrapheneCt/libAppSettings)** bridges to the Vita's own
system-software app-settings dialog, so a homebrew app can present a settings UI that looks and
behaves like the system's native one instead of building a custom options screen. Minimal
documentation upstream beyond a sample implementation — read that sample directly rather than
expecting a fuller written integration guide.

## PSVita Unity Utilities — for native Unity Vita builds, not ports

**[PSVita-Unity-Utilities](https://github.com/GlitcherOG/PSVita-Unity-Utilities)** is aimed at
building a Unity game *for* Vita from within the Unity Editor (VPK packaging, FTP deploy, build-and-
run, Vita remote control, TitleID/VitaDB-validation helpers) — **this is a different scenario from
everything else in this wiki**, which otherwise assumes you're bringing existing C/C++ source (a
library in [section 6](../06-porting-opengl-libraries-to-vitagl/README.md), a whole decompiled game
in [section 7](../07-porting-decompiled-games/README.md)) to the platform. Requires the ".NET 4.x
Equivalent" scripting runtime and Vita build target set to "PC Hosted" in Unity's build settings;
built and tested against Unity 2018.2.19f1, with unconfirmed compatibility for earlier versions like
2017. If your actual task is porting an *existing* Unity game whose scripting backend is already
compiled (Mono/AOT), that's [VitaMonoLoader](12-alternative-runtimes-openal.md) instead — a
different tool for a different stage of a Unity project's lifecycle.
