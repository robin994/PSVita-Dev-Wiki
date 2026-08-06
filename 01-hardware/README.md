# PS Vita Hardware

The Vita is a 2011 handheld built around a quad-core ARM Cortex-A9 SoC (with a genuinely separate
low-power core for background tasks), a PowerVR SGX543MP4+ GPU, and a memory architecture split
into several distinct physical pools that homebrew code has to be deliberate about — this is the
single biggest source of surprises for anyone coming from PC/desktop development, where "just
allocate more RAM" is rarely an option in the same way.

Two hardware revisions exist: the original **PCH-1000** (OLED screen, rear touchpad, physical
memory-card storage) and the **PCH-2000 "Vita Slim"** (LCD screen, thinner, internal flash instead
of a removable card slot in some regions). The **PlayStation TV (PSTV)** is a third variant — Vita
internals in a set-top-box form factor, no built-in screen or touch, HDMI output, and no rear
touchpad or motion sensors, which matters for any homebrew that assumes those inputs exist.

## Pages in this section

1. [Overview](01-overview.md) — SoC, screen, ports, the three hardware variants compared
2. [CPU architecture](02-cpu-architecture.md) — ARM Cortex-A9 cores, NEON, threading model
3. [GPU architecture](03-gpu-architecture.md) — PowerVR SGX543MP4+, tile-based deferred rendering, sceGxm
4. [Memory architecture](04-memory-architecture.md) — RAM tiers, CDRAM, physically-contiguous memory, budgets
5. [Storage & filesystem](05-storage-filesystem.md) — ux0/ur0/uma0/imc0, mount points, sceIo
6. [Input devices](06-input-devices.md) — touch, buttons, sticks, motion, camera, PSTV differences
7. [Multimedia hardware](07-multimedia-hardware.md) — H.264/AAC hardware decode, AVPlayer, audio DSP
8. [Networking](08-networking.md) — WiFi, sceNet stack, HTTP/SSL
9. [System software layers](09-system-software-layers.md) — kernel vs user, SceShell, taiHEN, the homebrew boot chain
