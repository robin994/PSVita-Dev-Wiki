# Reverse engineering & live debugging tools

[Debugging & tooling](07-debugging-tooling.md) covers the workflow this wiki's own porting work
actually uses day to day: `vitacompanion` for deploy/relaunch, `vita-parse-core` for **post-mortem**
crash analysis. That workflow has a real ceiling — a `.psp2dmp` coredump is a single frozen snapshot
at the moment of a fault, with only a handful of small memory windows captured (see that page's
"real gotcha" callout). It cannot show you a live sequence of events, let you set a breakpoint
before a suspected bad access, or single-step through the exact code path a bug takes. This page
covers the tools that *can* — live kernel debugging and disassembler/decompiler integration for the
Vita's binary formats — plus the broader disassembly/dumping toolset around them.

## Live kernel debugging: KVDB

**[KVDB](https://github.com/DaveeFTW/kvdb)** is a kernel-mode plugin that implements a
GDB-compatible debug stub for the Vita — the thing `vita-parse-core` fundamentally can't be, since
it only ever sees state *after* the fault already happened. With KVDB loaded and a GDB client
attached, you get what post-mortem analysis can't offer:

- Breakpoints set *before* a suspected bad access, rather than only ever seeing where execution
  already stopped.
- Single-stepping through the exact instruction sequence a bug takes.
- Live memory inspection while the app keeps running, not a handful of small pre-captured windows.

Loads via `config.txt` under `*KERNEL` (see [Kernel plugins & taiHEN](06-kernel-plugins-taihen.md)
for the mechanics of that file). Transport is UART by default; TCP/IP needs the companion `vdbtcp`
plugin, and the target application must already be running in debug mode before GDB connects —
KVDB doesn't handle launching the app itself.

**Known as alpha-quality, with limitations that matter for planning a debugging session**:
one debugging session per boot (plan for a console reboot between sessions, not just an app
relaunch), and memory faults aren't handled cleanly. Treat it as the tool to reach for when a bug
genuinely needs interactive stepping/breakpoints and a coredump's single frozen snapshot isn't
enough — not as a first-line replacement for `vita-parse-core`'s much lower setup cost.

## Disassembler/decompiler loaders

These bring a Vita `.elf`/SELF into a general-purpose reverse-engineering tool that already
understands ARM/Thumb disassembly, cross-references, and (for Ghidra/Binary Ninja) decompilation —
a meaningfully better environment for reading unfamiliar compiled code than raw
`arm-vita-eabi-objdump` output. This session's own workflow hit exactly that ceiling: confirming
VitaSDK's prebuilt `libz.a` uses real ARM NEON instructions in `inflate_fast` meant scrolling raw
`objdump -d` output by hand looking for `vld1`/`vst1` mnemonics — a proper loader with
cross-reference and decompiler views would have made that a much faster, more legible investigation
instead of grep-driven guesswork through disassembly text.

- **[GhidraVitaLoader](https://github.com/xerpi/GhidraVitaLoader)** — a Ghidra loader script for
  Vita ELFs. Optionally resolves imports/NIDs by loading VitaSDK's header set and a `db.yml` NID
  mapping file through the Script Manager after loading the binary; works without that step too,
  just with less symbol resolution. Install: drop `VitaLoader.java` plus the `yamlbeans` JAR into
  Ghidra's script/plugin paths.
- **[Vitaldr](https://github.com/LinkOFF7/vitaldr)** — an IDA Pro loader plugin for Vita ELF/SELF
  binaries (`eboot.bin`). Minimally documented upstream; treat the two IDA-loader options here as
  alternatives to compare directly against your installed IDA version rather than assuming either
  is current.
- **[IDA Game ELF Loaders](https://github.com/aerosoul94/ida_gel)** — a loader collection covering
  PS3, Vita, and Wii U ELFs in one project. Built and tested against IDA SDK 6.8 (Visual Studio
  2015) per its own README, with no stated compatibility claim for newer IDA versions — worth a
  quick compatibility check before committing to it over Vitaldr for a current IDA install.
- **[BinaryNinja-PSVitaLoader](https://github.com/computerman00/BinaryNinja-PSVitaLoader)** — loads
  Vita ELF/PRX2 binaries into Binary Ninja and resolves NID-named imports/exports back into real
  symbols in the default ELF view, with Vita-specific type definitions and cleanup for
  ARM/Thumb-mixed code. Tested against Binary Ninja 4.1.5902–4.2.6092; still needs manual handling
  for Thumb2 at binary startup per its own documented limitation.

Pick based on what you already have installed rather than switching tools for this specifically —
all three (Ghidra, IDA, Binary Ninja) get you past raw `objdump` text for genuinely complex
disassembly-reading work; none of the three loaders here claim to be strictly better than the
others across the board, and none has been evaluated against the others hands-on for this wiki.

## Broader dumping/extraction toolset

**[PSVita-RE-tools](https://github.com/CelesteBlue-dev/PSVita-RE-tools)** is a bundle (FAPS Team,
GPLv3) covering adjacent RE needs beyond disassembly:

- **Logging**: `PrincessLog`, `PSMLogUSB` (obsolete `ShipLog 2.0` alternative also included).
- **Decompilation**: `VitaDecompilerMod` (binaries → pseudo-C with exports), `prxtool` (assembly-level).
- **SELF/ELF conversion**: `vita-unmake-fself` (decrypt SELF → ELF), `vita-elf-inject` (the reverse,
  reinserting modified code into a SELF).
- **On-device decryption**: `FAGDec` decrypts game/system modules on the device itself.
- **Kernel/physical memory**: `Kdumper` (kernel dumps, TestKit/DevKit only), `physmem_dumper`,
  `noASLR` (disable ASLR for reproducible addresses across runs — directly useful alongside KVDB or
  a coredump workflow where a fixed load address simplifies address-rebasing math).
- **Firmware/NID extraction**: `psp2-kernel-bootimage-extract`, `psp2-kbl-elf-extract`,
  `psp2-syslibtrace-nids-extract`.
- **`ioPlus`** — a kernel plugin elevating userland I/O permissions.

**[RegistryEditorMOD](https://github.com/devnoname120/RegistryEditorMOD)** reads and edits every
setting in the Vita's system registry (the config database backing most system-level settings, not
a Windows-style registry) through an organized tree view, with position memory added over the
original app it forked from. No safety warnings ship with the tool itself — treat it with the same
caution as any direct system-settings editor, since incorrect edits can affect system stability.

**[MaiDumpToolEN](https://github.com/LioMajor/MaiDumpToolEN)** is an English translation of a game
dumper/patcher tool. Its upstream documentation is minimal enough that this wiki can't responsibly
describe its internals beyond that one-line description — if you need it, expect to read the
tool's actual behavior from source/testing rather than its README.

**CXML-Decompiler** (listed at `silica.codes/SilicaAndPina/cxml-decompiler`, redirecting to
`git.silica.codes/SilicaAndPina/cxml-decompiler`) is described only as a "PlayStation Mobile
decompiler" — PSM (PlayStation Mobile) being the separate, no-longer-supported Sony platform covered
in [Alternative toolchains & deployment](11-alternative-toolchains-deployment.md#psm-reborn--playstation-mobile-archive-historical-only).
**The self-hosted git instance returned unrelated/garbled content when fetched for this page** (a
mix of unrelated text fragments, not a real README) — this wiki can't verify anything about the
tool's actual internals beyond that one-line description. If you're doing PSM-era decompilation
work specifically, verify the repo is still live and reachable before relying on it.

## Reference material (not tools, but worth knowing exist)

- **[PS Vita Decrypted Firmwares](https://files.olebeck.com/firmware/vita/decrypted)** — a
  collection of decrypted firmware images, useful when a disassembler loader or NID-resolution
  workflow needs ground truth for system module versions across firmware revisions.
- **[debugging.games](https://dg.getrektby.us/Playstation%20Vita)** — Vita game binaries that
  retain debug symbols; a useful "known good" comparison target when practicing/validating a
  disassembler-loader or NID-resolution setup against a binary where the real function names are
  already known, rather than starting cold on a fully-stripped target.
- **[tcrf.net's PS Vita page](https://tcrf.net/PlayStation_Vita)** — documents unused/prototype
  console-level features, not individual games; tangential to day-to-day porting work but
  occasionally relevant if a system behavior seems undocumented and might be a leftover/disabled
  feature rather than a bug in your own code.
