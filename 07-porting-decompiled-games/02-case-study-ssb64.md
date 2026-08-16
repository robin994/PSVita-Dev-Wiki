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

## v1 scope: 60 fps + widescreen, no netplay

Confirmed target: a working v1 at a stable 60 fps with widescreen, netplay explicitly deferred to
v2. Both v1 features already exist upstream — this is a config-and-tune job, not new engineering:

- **Widescreen** (`port/widescreen/`) is a CVar-gated system already handling viewport, projection,
  and HUD-anchoring for 16:9. Vita's 960×544 is near-16:9. v1 work: flip the CVar on, confirm the
  three affected planes (viewport / projection / HUD anchors) render correctly through vitaGL — not
  building widescreen support.
- **60 fps** (`port/interpolation/`) already runs game logic at a fixed, audio-locked 60 Hz tick;
  render interpolation only *fans out* to multiples (120/180/240) above that. v1 needs none of the
  fan-out machinery — target the system's own default single-tick-per-render path (`k=1`) and put
  all real effort into hitting the 16.67 ms/frame budget through standard vitaGL tuning (see
  [03-vitagl/07-performance-best-practices.md](../03-vitagl/07-performance-best-practices.md)):
  draw-call count, texture budget, resolution. This is the one genuinely open-ended risk in v1 — a
  fighting engine with several simultaneous characters and particles is a heavier ask than
  Ghostship/SpaghettiKart's scenes were.
- **Netplay is out of scope for v1**, full stop — don't design around it yet. The project's rollback
  architecture doc is the right starting point when v2 picks it up.

## Recommended sequence

1. Build and play the existing desktop port first — the only working baseline once things start
   rendering wrong on-device. Confirm widescreen + interpolation both work there with defaults.
2. Fork; add a `Makefile.vita` templated from SpaghettiKart's (the closest real precedent),
   adapting `TARGET`/`TITLE`/`GAME_SOURCES` to this game's `decomp/src/` + `port/` layout.
3. Link the existing libultraship OpenGL/SDL2 backends against vitaGL/vitashark/VitaSDK's SDL2
   as-is first; only build a dedicated platform shim if that genuinely fails to compile or run.
   Confirm with Rinnegatamante what his actual Vita-side libultraship diff contains before assuming
   one is needed at all.
4. Iterate to a link — standard Vita-port loop.
5. LiveArea assets + `vita-elf-create`/`vita-make-fself`/`vita-pack-vpk` packaging.
6. On-device bring-up: boot, controller mapping, audio.
7. Enable the widescreen CVar; fix whatever doesn't map cleanly to 960×544.
8. GPU performance triage against real frame-time numbers until 60 fps holds. This is where most
   of the v1 calendar time goes.
9. **v2, not now:** wire a Vita `sceNet` transport into the existing rollback-netcode boundary.
