# Porting OpenGL-Based Libraries to vitaGL

This section is about a specific, recurring task: taking an existing desktop/cross-platform library
that already expects to render through an OpenGL-shaped API (a UI toolkit, a game engine) and
getting it running through [vitaGL](../03-vitagl/README.md) instead. This is **not** the same
problem as building a new Vita app from scratch — see [vita2d](../05-vita2d/README.md) and
specifically [vita2d vs vitaGL](../05-vita2d/07-vita2d-vs-vitagl.md) for when porting isn't the
right tool at all.

## Pages in this section

1. [General methodology](01-methodology.md) — the principles extracted from a real, shipped port
   ([imgui-vita](../04-imgui/02-imgui-vita-backend.md)), generalized beyond that one library
2. [Case study: RmlUi](02-case-study-rmlui.md) — the same methodology applied as a *planning*
   exercise to a library that has **not** actually been ported to Vita yet — read this as a worked
   example of the planning process, not as a description of working code

## Scope and honesty about sourcing

Page 1 is grounded in a real, existing port (imgui-vita) — verified by reading its actual source
against upstream Dear ImGui. Page 2 is explicitly **not** that: RmlUi has no known Vita/vitaGL port
at the time of writing, so that page is deliberately-labeled planning guidance (a roadmap with
verifiable milestones), not a description of a working implementation. Don't treat it as more
settled than that.
