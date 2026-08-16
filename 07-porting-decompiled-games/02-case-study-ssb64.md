# Case Study: Super Smash Bros. 64 (planning stage)

**Status: planning exercise, not a working port.** This page applies
[the methodology](01-methodology.md) to a fourth Shipwright-family game that has not been ported to
Vita at the time of writing. Treat it as a roadmap, not a description of running code — same caveat
as section 06's RmlUi case study.

The full analysis this page summarizes was written up as a standalone report; see
`ssb64-vita-port-strategy.html` for the complete version with a build-system comparison table and
a phased plan.

## What the project actually is

The upstream repo is `JRickey/BattleShip` — a name with nothing to do with the board game. It's a
PC port of **Super Smash Bros. 64**, built from a complete decompilation
(`ssb-decomp-re`), using the same libultraship + Torch stack as every other game in this family.
Code-naming a decompilation port after something unrelated is standard practice in this scene — the
same reason the Zelda 64 port in this wiki's other examples is called "Ghostship" and the Mario Kart
64 port is called "SpaghettiKart." No ROM or copyrighted asset ships in the repo or in any port;
the end user supplies their own legally-dumped ROM at first run, same as every decomp-based
homebrew already in the NeoVitaDB catalog.

## Why the methodology applies directly

The repo's own structure already matches the pattern in page 1 almost exactly: a `libultraship/`
submodule, a `Torch/` submodule, a `port/` directory for the C++ port layer, and `yamls/` for the
asset extraction config. The project's own documentation explicitly names **SpaghettiKart** and
**Starship** (a Star Fox 64 port) as its architectural reference ports — SpaghettiKart being not
just architecturally similar but Rinnegatamante's own already-completed Vita port, making it the
closest thing to a literal template available. (See page 1's correction note: there's no verified
evidence of a dedicated Vita platform shim to hunt for here — the real work is more likely
concentrated in the `Makefile.vita` and which existing libultraship backend it links against.)

## Where this game is harder than the existing reference ports

Two things distinguish it from Ghostship/SpaghettiKart/2ship2harkinian:

- **Real-time performance budget.** A fighting-game engine with several simultaneous characters,
  physics, hitboxes, and particle effects asks more of the GPU per frame than a kart racer's mostly-
  static tracks or an adventure game's individual rooms. Expect the same triage every demanding
  vitaGL port needs (see [03-vitagl/07-performance-best-practices.md](../03-vitagl/07-performance-best-practices.md)):
  resolution scale-down, draw-call and texture budget cuts, possibly disabling some effect layers.
- **Netplay.** The project has its own rollback-netcode architecture document. None of the three
  reference ports had to solve online play. A Vita transport means real `sceNet` integration — a
  substantial, separable piece of work with no existing template in this family, reasonable to scope
  as a post-launch milestone rather than part of initial bring-up.

Everything else — the decomp tree being larger than a single-game port's `GAME_SOURCES` list, the
mechanical link-and-fix loop — is more volume, not more novelty, and is what page 1's "link-and-fix
loop dominates the timeline" best practice already predicts.

## Recommended sequence (summary)

1. Build and play the existing desktop port first — the only working baseline once things start
   rendering wrong on-device.
2. Fork; add a `Makefile.vita` templated from SpaghettiKart's (the closest real precedent),
   adapting `TARGET`/`TITLE`/`GAME_SOURCES` to this game's `decomp/src/` + `port/` layout.
3. Try linking the existing libultraship OpenGL/SDL2 backends against vitaGL/vitashark/VitaSDK's
   SDL2 as-is first, and only build a dedicated platform shim if that genuinely doesn't compile or
   run — confirm with Rinnegatamante directly what his actual Vita-side libultraship diff looks like
   before assuming one is needed.
4. Iterate to a link — standard Vita-port loop.
5. LiveArea assets + `vita-elf-create`/`vita-make-fself`/`vita-pack-vpk` packaging.
6. On-device bring-up: boot, controller mapping, audio, then GPU performance triage against real
   frame-time numbers.
7. Netplay, as a stretch goal once local play is solid — wire a Vita `sceNet` transport into the
   rollback-netcode boundary the project's own architecture doc already defines.
