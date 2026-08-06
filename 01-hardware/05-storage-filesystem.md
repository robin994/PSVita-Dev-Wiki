# Storage & Filesystem

## Mount points, not drive letters

The Vita's filesystem is exposed as a set of named mount points, each backed by different physical
or virtual storage, roughly analogous to Unix mount points more than Windows drive letters (despite
the `x0:` naming resembling the latter). The ones homebrew developers actually touch regularly:

- **`ux0:`** — the memory card (original hardware) or internal storage partition used the same way
  (later hardware/CFW setups). This is where installed applications, save data, and — critically for
  homebrew — most persistent app data lives. `ux0:data/<AppName>/` is the idiomatic place for a
  homebrew app's own config/cache/save files, mirroring how `ux0:app/<TITLEID>/` holds an installed
  app's own package contents.
- **`ur0:`** — a small dedicated system partition on the *internal* flash, present even when the
  memory card is removed/full. Historically used by some homebrew (and required by certain CFW
  components) for things that must survive even without a memory card inserted, though most
  full-sized app data still belongs on `ux0:`.
- **`uma0:`** — represents whatever's plugged into the multi-use port when it's a storage device
  (some official peripherals; also used by certain homebrew/CFW installer flows).
- **`imc0:` / `xmc0:`** — older/alternate memory-card mount naming depending on firmware/hardware
  revision; less commonly referenced directly in modern homebrew, which mostly targets `ux0:`.
- **`pd0:`** — an internal partition CFW infrastructure (like `taiHEN`) uses for its own
  configuration; not a general-purpose app storage target.
- **`host0:`** — only meaningful in specific bootstrap/development contexts (e.g. some CFW exploit
  chains temporarily redirect reads here); not something a normal installed app reads from.
- **`app0:`** — the *running application's own* read-only package contents (resembles `ux0:app/<own
  TITLEID>/` but is the correct, portable way to reference "my own bundled assets" without hardcoding
  a title ID). This is where a VPK's bundled data files end up accessible from at runtime.

## What this means practically

- **Persistent user data belongs on `ux0:data/<YourAppName>/`.** Don't write into `app0:` (it's
  read-only at runtime anyway) or assume `ur0:` has meaningful free space for anything but small
  config files.
- **Always create your data directory defensively** (`sceIoMkdir`, ignoring the "already exists"
  error) rather than assuming it exists — a fresh install has nothing there yet.
- **Storage can be genuinely absent or full.** Memory cards can be removed (on original hardware) or
  simply full; check `sceIoGetstat`/write return codes rather than assuming writes always succeed,
  especially for anything a user might reasonably run without a memory card inserted at all.

## The filesystem API: sceIo

Vita homebrew talks to storage through `sceIo*` functions (`sceIoOpen`, `sceIoRead`, `sceIoWrite`,
`sceIoClose`, `sceIoGetstat`, `sceIoMkdir`, `sceIoRemove`, `sceIoDopen`/`sceIoDread` for directory
listing, and so on) — a POSIX-*flavored* but not POSIX-*identical* API. VitaSDK's newlib layer maps
standard C `fopen`/`fread`/etc. onto these under the hood for portable code, but be aware:

- Paths always include the mount-point prefix (`"ux0:data/MyApp/save.dat"`), there's no concept of
  a single-rooted `/` filesystem spanning all mount points.
- `sceIoGetstat` fills a `SceIoStat` struct (mode, size, timestamps) — the idiomatic way to check
  "does this exist" before acting on it, since there's no separate lightweight `access()`-equivalent
  most homebrew reaches for instead.
- File I/O crossing threads needs the same care as any other shared-resource access — see
  [CPU architecture](02-cpu-architecture.md) for the threading/synchronization primitives available.

## FIOS2

Sony's higher-level asynchronous file I/O layer (`SceFios2`) exists for cases needing background/
prioritized/batched file access without hand-rolling a thread pool around `sceIo*` calls directly.
Most simple homebrew doesn't need it and sticks to synchronous `sceIo*`/stdio calls on a dedicated
worker thread instead, but it's worth knowing it exists if you're building something with heavier
streaming I/O needs (large asset streaming, for instance).

## FAT/exFAT and practical limits

Memory cards are formatted FAT32/exFAT depending on card size and firmware version, which brings
the usual FAT-family constraints along with it (case-insensitive-but-case-preserving behavior in
practice, individual file size ceilings under plain FAT32 on some setups, fragmentation behavior
under sustained heavy writes). Nothing homebrew developers typically hit in practice unless writing
very large single files, but worth knowing if you ever see a write mysteriously cap out.
