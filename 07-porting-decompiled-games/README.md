# Porting Decompiled Console Games to PS Vita

This section covers a different kind of port from [section 06](../06-porting-opengl-libraries-to-vitagl/README.md):
not a single library being retargeted onto vitaGL, but a **whole decompiled game** being brought to
the Vita — the pattern behind community ports like Rinnegatamante's Ghostship (Zelda 64: OoT/MM),
SpaghettiKart (Mario Kart 64), and 2ship2harkinian (Zelda 64: Majora's Mask).

These are not emulation. They're clean-room decompilation projects (matched C reconstructed from
the original console binary, with the community project verifying byte-identical reassembly)
recompiled to run natively, against a portability layer that stands in for the original console's
SDK. The Vita port only has to deal with that portability layer — the decompiled game code itself
never changes.

## Pages in this section

1. [Methodology: the libultraship/Shipwright pattern](01-methodology.md) — the three-layer
   architecture (decomp / libultraship / asset pipeline) that makes this class of port tractable,
   and the concrete build-system recipe verified across three real, shipped Vita ports
2. [Case study: Super Smash Bros. 64](02-case-study-ssb64.md) — the same methodology applied as a
   *planning* exercise to a fourth game in this family that has not been ported to Vita yet

## How this differs from section 06

Section 06 is about taking a library that already speaks OpenGL and getting *that library* running
through vitaGL. This section is about a full game engine where the game code itself has zero
platform awareness — every N64-SDK-shaped call it makes is already routed through a portability
layer (libultraship) on every platform, desktop included. Porting to Vita means extending that one
layer, not touching the several-hundred-thousand-line decompiled game above it. The actual
low-level graphics work — vitaGL, shader precompilation via vitashark/SceShaccCgExt — is the same
foundation [section 03](../03-vitagl/README.md) already documents; this section is about the extra
scaffolding a full game (as opposed to a single library) needs around that foundation.

## Scope and honesty about sourcing

Page 1 is grounded in three real, shipped Vita ports, verified by reading their actual
`Makefile.vita` files and repository structure (not the game's copyrighted source or assets — none
of these projects distribute ROM data, and neither does this wiki). Page 2 is explicitly a planning
exercise for a game with no existing Vita port at the time of writing — treat it as a roadmap with
verifiable milestones, not a description of working code, same caveat as section 06's RmlUi case
study.
