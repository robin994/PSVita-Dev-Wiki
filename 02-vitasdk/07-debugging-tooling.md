# Debugging & Tooling

There's no host-native build for most Vita homebrew (see
[Project setup & build system](02-project-setup-build-system.md)) — you're always cross-compiling
for the device, which shapes the whole debugging workflow around "get feedback from the actual
console" rather than a local desktop debug loop.

## Deploying builds to a device

**VitaShell** (see [Hardware: system software layers](../01-hardware/09-system-software-layers.md))
is the standard tool for getting a freshly built `.vpk` onto a real device — its built-in FTP server
lets you push files from your dev machine directly onto the console's storage over the local
network, which is the fastest iteration loop for "build, transfer, install, test" without physically
moving a memory card back and forth.

### Faster than reinstalling the VPK every time

Reinstalling the whole `.vpk` through VitaShell on every build is slow enough to actively discourage
tight iteration. Two things speed this up, usable independently:

- **Replace `eboot.bin` directly** instead of reinstalling: once a VPK is installed once, its
  executable lives at `ux0:app/<TITLEID>/eboot.bin` — FTP-overwriting that file (VitaShell's FTP
  server, or any FTP client/`curl -T`) and relaunching the app picks up a new build without going
  through the installer again. See [Homebrew app anatomy](03-homebrew-app-anatomy.md) for how the
  title ID maps to that path.
- **[`vitacompanion`](https://github.com/devnoname120/vitacompanion)** — a taiHEN plugin pair (one
  kernel module, one user module) that keeps a background FTP server and a remote command listener
  running on the Vita at all times, independent of whatever homebrew is currently running. Combined
  with the `eboot.bin`-replace trick above, a single CMake custom target (or, for a hand-rolled
  Makefile build with no CMake target system, just the three commands run directly from a shell) can
  close the running app, push the new build, and relaunch it in one step:

  ```cmake
  set(PSVITAIP "192.168.0.198" CACHE STRING "PSVita IP (for FTP access)")
  add_custom_target(send
                    COMMAND echo destroy | nc ${PSVITAIP} 1338
                    COMMAND curl -T eboot.bin ftp://${PSVITAIP}:1337/ux0:/app/${VITA_TITLEID}/
                    COMMAND echo launch ${VITA_TITLEID} | nc ${PSVITAIP} 1338
                    DEPENDS ${VITA_VPKNAME}.vpk-vpk
                    )
  ```

  Run with `cmake --build build --target send`. Or, without CMake, the same three steps as plain
  shell commands (works from any build system, including a standalone `Makefile.vita`):

  ```sh
  echo destroy | nc -w3 $VITAIP 1338
  curl -T eboot.bin ftp://$VITAIP:1337/ux0:/app/$TITLEID/eboot.bin
  echo "launch $TITLEID" | nc -w3 $VITAIP 1338
  ```

  ### Build & install

  ```sh
  git clone https://github.com/devnoname120/vitacompanion
  cd vitacompanion && mkdir build && cd build
  cmake .. && make
  ```

  Copy both `vitacompanion.suprx` and `vitacompanion_kernel.skprx` to `ur0:tai/` (VitaShell → SELECT
  to start its FTP server, or any FTP client), then add both to `ur0:tai/config.txt`:

  ```
  *KERNEL
  ur0:tai/vitacompanion_kernel.skprx

  *main
  ur0:tai/vitacompanion.suprx
  ```

  The kernel module must be loaded — the user module imports its input-simulation API from it and
  doesn't load it on its own. Reboot after installing or replacing either module so no older copy is
  left resident. To run two side-by-side copies (e.g. a stable one plus a build under test) without
  the second clobbering the first, build with distinct ports/module name:
  `cmake .. -DVITACOMPANION_FTP_PORT=1340 -DVITACOMPANION_CMD_PORT=1341
  -DVITACOMPANION_MODULE_NAME=vitacompanion_test`.

  ### FTP server (port `1337`)

  Vita-style paths (`ux0:/somedir/`) and FTP-absolute paths (`/ux0:/somedir/`) both work; standard
  `PASV`/`EPSV`/`PORT`/`EPRT`, ASCII/binary transfer, and resume/append are all supported, so any
  generic FTP client works, not just `curl`. To retrieve a file (e.g. pulling a `.psp2dmp` — see
  below — without first copying it into an app's own data directory):

  ```sh
  curl -s -o local.psp2dmp ftp://$VITAIP:1337/ux0:/data/psp2core-....psp2dmp
  curl -s ftp://$VITAIP:1337/ux0:/data/          # directory listing
  ```

  ### Command server (port `1338`) — full reference

  One command per line (or `;`-separated for a chain, trailing `;` optional), sent over a raw TCP
  connection (`nc $VITAIP 1338` or equivalent):

  | Command   | Arguments                      | Effect |
  | --------- | ------------------------------- | ------ |
  | `help`    | —                                | list commands |
  | `version` | —                                | print the loaded module's version |
  | `destroy` | —                                | kill all running applications |
  | `launch`  | `<TITLEID>`                      | launch an installed app by title ID |
  | `kill`    | `<TITLEID>`                      | kill one specific app by title ID |
  | `reboot`  | —                                | reboot the console |
  | `screen`  | `on` / `off`                     | turn the display on or off |
  | `nosleep` | `on` / `off` / `status`          | toggle automatic-suspend prevention (on by default at boot) |
  | `press` / `release` | button, stick, or touch target | synthetic input injection (below) |
  | `wait`    | duration, e.g. `500ms`, `3s`     | delay before the next chained command |

  Synthetic input is genuinely useful for scripted, repeatable UI-flow testing (navigating a menu the
  same way on every run rather than doing it by hand each time) without needing to touch the console:

  ```sh
  # Buttons
  echo 'press cross; wait 100ms; release cross' | nc $VITAIP 1338
  # Analog sticks - coordinates are 0-255, 128 = center
  echo 'press left-stick 0 128; wait 200ms; release left-stick' | nc $VITAIP 1338
  # Touch - 4 independently tracked slots (0-3), raw 1920x1088 touch space;
  # repeating `press` on an active slot moves that touch, preserving its contact ID
  echo 'press front-touch 0 960 544; release front-touch 0' | nc $VITAIP 1338
  echo 'release all' | nc $VITAIP 1338   # clear every synthetic input at once
  ```

  Button names: `select`, `start`, `up`/`right`/`down`/`left`, `l`/`r`/`l1`/`r1`/`l2`/`r2`/`l3`/`r3`,
  `triangle`, `circle`, `cross` (`x` also works), `square`, `ps`.

## Real-time on-device logging

Standard `printf`-to-a-terminal debugging doesn't exist in the usual sense — there's no attached
console the way a desktop app has stdout. Two complementary approaches cover most needs:

- **File-based dumps**: write diagnostic state to a file on `ux0:` (`sceIoOpen`/`sceIoWrite`) at
  key points, then retrieve it afterward via VitaShell/FTP. Simple, reliable, zero extra
  dependencies — the right default for "capture state at a specific failure point" debugging. The
  downside is it's inherently after-the-fact — you get a snapshot, not a live stream, so it's weaker
  for understanding *timing*/*ordering* of events across threads.
- **Real-time UDP logging** (e.g. the community `debugnet` library,
  `github.com/psxdev/debugnet`): the app calls `debugNetInit(pc_ip, port, level)` once at startup
  and `debugNetPrintf(level, fmt, ...)` anywhere you'd otherwise reach for a desktop `printf`; a
  listener on the dev machine (as simple as `socat udp-recv:<port> stdout`) prints each line live as
  it arrives. This is genuinely valuable for anything involving timing, ordering, or
  cross-thread/cross-callback interaction — e.g., understanding what order a sequence of SDK
  callbacks actually fires in on real hardware — in a way a post-hoc file dump can't show you as
  clearly. **Treat it as temporary instrumentation**: wire it in for an investigation, pull it back
  out (the library dependency, the init call, every call site) once you're done, rather than leaving
  live network logging permanently built into a shipped release.
- **`psp2shell`** is a similar, somewhat heavier-weight alternative some projects use instead of
  `debugnet` — same broad idea (live diagnostic output over the network to a PC-side listener), a
  bigger setup footprint in exchange for more built-in functionality (remote command execution,
  not just one-way logging).

## Post-mortem crash dump analysis (`.psp2dmp`)

When an app crashes on real hardware, the system can write a `.psp2dmp` file (named
`psp2core-<pid>-<addr>-<eboot-name>.psp2dmp`) into the crashed app's data directory — pull it off
via VitaShell/FTP the same way you'd retrieve any other on-device file. It's a gzip-compressed ELF
**core file**, but not a standard Linux-style core: it's Sony's own `SceCoredump` note layout
(`PT_NOTE` segments named `MODULE_INFO`, `THREAD_INFO`, `THREAD_REG_INFO` rather than the usual
`NT_PRSTATUS`), so generic tools (`gdb target core`, standard `readelf -n` interpretation) can't
make sense of the register/thread state out of the box — you get "Couldn't find general-purpose
registers in core file" from gdb even though the data is right there.

**[`vita-parse-core`](https://github.com/xyzz/vita-parse-core)** is the community tool that
understands this layout: `python2 main.py core_file.psp2dmp your_app.elf` (needs `pyelftools==0.24`,
Python 2). Two practical gotchas going in:
- It targets Python 2 and a pinned ancient `pyelftools`; on a modern Python 3 environment the exact
  pinned version won't import (`collections.MutableMapping` was removed), and even patching past
  that hits more py2-only syntax (`string.letters`, `xrange`, byte/str handling). The note format
  itself is simple enough (three fixed-layout `PT_NOTE` blobs, documented in the tool's own
  `core.py`) that reimplementing just the parsing logic in Python 3 against a current `pyelftools`
  is less friction than fighting the environment mismatch.
- Pass the **unstripped `.elf`** the build produced (build it with `-g` for line info) — not the
  `.velf` and not the final `eboot.bin`. Those are Vita-specific converted/signed formats standard
  ELF tooling (`addr2line`, `nm`) can't read directly.

**The real gotcha, the one that silently gives you nothing rather than an error**: the crash dump's
`MODULE_INFO` note reports each module's code segment by its *runtime* load address (e.g. your
main executable's code segment starting at `0x81054000` on that particular boot) — this is **not**
the address your `.elf` was linked at. Feeding `arm-vita-eabi-addr2line` the raw
`(crash_pc - segment.start)` offset resolves to nothing (`??`, `??:0`) with no error to tell you
why. You have to add back the ELF's *own* link-time base address for its code `LOAD` segment
(`arm-vita-eabi-readelf -lW your.elf` → the `VirtAddr` of the `R E` segment, typically `0x81000000`
for a default VitaSDK link) before handing the address to `addr2line`:

```
elf_address = ELF_LOAD_SEGMENT_VIRTADDR + (crash_runtime_address - module_segment.start)
arm-vita-eabi-addr2line -e your.elf -f -C -a <elf_address>
```

Get this offset wrong and every single lookup fails silently rather than erroring — it's easy to
conclude the dump is unusable when actually just one add is missing. Once addresses are rebased
correctly, `THREAD_REG_INFO` gives you full GPRs (R0–R12, SP, LR, PC) per thread, letting you
resolve the crashing PC and LR to exact function + source line, and walk the stack (each 4-byte
slot near SP, resolved the same way) for a lightweight backtrace even without a live debugger
session.

## Emulator (Vita3K) vs real hardware

**Vita3K** is the community Vita emulator, genuinely useful for a fast local iteration loop (no
physical device round-trip needed for most testing), but it is **not behaviorally identical to real
hardware**, and treating "works in the emulator" as proof of correctness is a real, recurring source
of shipped bugs. The most concrete, well-documented gap: the emulator's software H.264 decoder does
not enforce the same profile/level ceiling the real hardware decoder does — content that plays fine
in Vita3K can fail outright on a real console (see
[Hardware: multimedia hardware](../01-hardware/07-multimedia-hardware.md)). More generally, any API
whose real hardware behavior comes from years of reverse-engineered, partially-documented
understanding (see [Overview & toolchain](01-overview-toolchain.md)) is a candidate for an emulator
that models the *documented* behavior faithfully while the *real* console does something subtly
different (an event ID that fires differently, a resource limit the emulator doesn't model at all).

**Practical rule**: use Vita3K for fast day-to-day iteration, but verify anything touching hardware-
adjacent behavior (media decode, precise timing, resource-limited subsystems like the
physically-contiguous memory pool) on a real device before considering it done — and if you only
have emulator access for a given change, say so explicitly rather than silently presenting
emulator-only verification as equivalent to real-hardware verification.

## Diagnosing "it works sometimes" / leak-shaped bugs

A useful general pattern for this platform, given the constrained/pooled memory model (see
[Hardware: memory architecture](../01-hardware/04-memory-architecture.md)): symptoms like "works the
first time, fails the second time" or "works at low settings/resolution but not high" often point at
a resource-pool exhaustion issue (a callback-based deallocation contract that isn't reliably
honored on every code path, accumulating leaked allocations across repeated use) rather than a
straightforward logic bug. When you see that shape of symptom, checking whether something is
tracking and force-releasing its own allocations on every exit path — rather than trusting an SDK
callback to always fire — is a productive early hypothesis.
