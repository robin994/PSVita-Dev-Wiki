# UX Patterns for Handheld/Controller-First Apps

Dear ImGui's defaults and most of its ecosystem's example code assume a desktop mouse+keyboard
user. Building a genuinely comfortable UI on the Vita means deliberately designing around a
different input model — this page is about UX judgment calls, not API mechanics (see
[Input handling](03-input-handling.md) for the mechanics).

## Pick one primary interaction model and commit to it

Mixing touch, stick-as-mouse pointer emulation, and gamepad navigation all active at once in the
same UI tends to feel incoherent — a user reasonably expects consistent behavior from whichever
input method they're using at a given moment, and Dear ImGui's own visual affordances (focus
highlight rectangles, hover states) are themselves generally designed around *one* coherent
interaction model at a time. Decide upfront: is this primarily a touch-driven app (natural for
something explicitly designed to be tapped, since the Vita's front panel genuinely is a
touchscreen), or a controller-first app (natural for anything meant to feel at home alongside
actual games, or for PSTV, which has **no touch capability of any kind at all** — see
[Hardware: input devices](../01-hardware/06-input-devices.md), meaning any touch-*only* design
locks out PSTV entirely). Many well-regarded homebrew apps land on **gamepad-navigation-first, with
touch as a nice-to-have secondary path** specifically because it degrades gracefully to PSTV without
a redesign.

## Design for large, clearly-focusable targets

Gamepad navigation moves a focus highlight between discrete widgets rather than a free-roaming
pointer — this works best with a UI made of clearly-delineated, reasonably-sized interactive
elements arranged in a layout that has an obvious, predictable navigation order (grid layouts and
simple vertical lists navigate intuitively; deeply nested or irregularly-arranged widget layouts can
produce confusing or seemingly-random focus jumps under D-pad navigation). If you're designing a
grid of items (an icon browser, a list of installable apps), keep the layout's actual arrangement
consistent with how you'd want D-pad up/down/left/right to move between items — Dear ImGui's nav
system infers movement from screen-space widget positions, so an unusual layout produces unusual-
feeling navigation.

## Respect regional face-button convention

As noted in [Hardware: input devices](../01-hardware/06-input-devices.md), circle-vs-cross for
confirm/cancel is a real regional UX convention Sony's own system UI honors via a system setting.
Homebrew that wants to feel native/polished should read that setting and adapt its own
Activate/Cancel button mapping to match, rather than hardcoding one convention — a detail easy to
overlook but noticeable to users on the "wrong" side of the convention for a hardcoded choice.

## Avoid modal dead-ends in a controller-only flow

Any dialog, popup, or confirmation prompt needs a clear, reachable way to dismiss/cancel it using
whatever input model is active — a touch-designed confirmation dialog with a small "Cancel" target
that's easy to tap but has no sensible gamepad-focus path is a real, frustrating dead end for a
controller-first user. Test every modal/dialog flow specifically under gamepad navigation, not just
however you personally tested it during touch-based development.

## Text entry is a real UX design point, not just an implementation detail

Since text entry routes through a modal system dialog (`sceImeDialog` — see
[Input handling](03-input-handling.md)) rather than an inline ImGui text field the user types
directly into, the UX flow is inherently "tap/select a field → system keyboard overlay opens → type
→ confirm → overlay closes, field now shows the result" rather than the immediate, no-modal-
interruption feel of typing directly into a desktop text box. Design search bars, name-entry fields,
and similar UI with this modal round-trip in mind (clear visual affordance that a field is
"launch-a-dialog" rather than "type directly here," since the interaction genuinely differs from
what a user might expect from visual similarity to a normal-looking text box alone).

## Anti-burn-in and long-session considerations

OLED-panel Vita units (original PCH-1000) carry the same long-static-content burn-in risk any OLED
display does. Homebrew designed to stay open on one relatively static screen for extended periods
(a downloader progress screen, an idle menu) is worth considering some mitigation for (subtle
periodic UI shifting, dimming, or similar) — a detail easy to overlook when developing/testing in
short bursts but relevant for how the app is actually used in practice.

## Summary

- Pick one primary interaction model; don't leave every input source on by default expecting users
  to figure out a consistent experience themselves.
- Design layouts with predictable gamepad-navigation flow in mind, not just visual arrangement.
- Respect the platform's regional confirm/cancel button convention.
- Test every modal/dialog under gamepad-only input specifically.
- Treat text entry's modal system-dialog round-trip as a real UX design constraint, not an
  implementation footnote.
- Consider burn-in mitigation for long-session, static-content screens on OLED hardware.
