# Hardware Overview

## The SoC

The Vita runs on a custom SoC built around:

- **CPU**: quad-core ARM Cortex-A9 (ARMv7-A, 32-bit), clocked around 444 MHz stock (homebrew CFW
  commonly overclocks to ~500 MHz total; the retail firmware itself reserves clock headroom it
  rarely uses). Homebrew typically only gets **3 of the 4 cores** — the fourth is reserved by the
  system for background/low-power tasks in most configurations.
- **GPU**: PowerVR SGX543MP4+ (4 cores), clocked around 222 MHz stock. Tile-based deferred
  renderer — see [GPU architecture](03-gpu-architecture.md).
- **RAM**: 512 MB total system RAM, of which roughly 128 MB is reserved for the system
  (OS/firmware/background services), leaving on the order of ~300+ MB usable by a foreground app —
  the exact usable figure varies by firmware version and what system features are active. A
  separate ~128 MB of **VRAM (CDRAM)** exists specifically for graphics use.
- **Front touchscreen**: 5-inch, 960×544 native resolution (the number every 2D/UI layout on the
  Vita is built around), capacitive multi-touch.
- **Rear touchpad**: capacitive multi-touch, no visual display behind it — present on both Vita
  revisions, **absent on PSTV**.

## The three hardware variants

| | PCH-1000 | PCH-2000 ("Slim") | PSTV |
|---|---|---|---|
| Screen | 5" OLED | 5" LCD | none (HDMI out) |
| Rear touchpad | yes | yes | no |
| Storage | Vita memory card (proprietary) | memory card or internal flash (region-dependent) | memory card |
| Motion sensors | accelerometer + gyroscope | accelerometer + gyroscope | none |
| Rear camera | yes | yes | no |
| Battery | yes (handheld) | yes (handheld, smaller) | none (mains-powered) |
| Ports | one proprietary multi-use port | same | full-size USB, HDMI, Ethernet on some models |

Code that assumes touch, motion, or a camera exist will misbehave on PSTV unless it checks first
(`sceTouchGetPanelInfo`, motion sensor init return codes, etc. all fail gracefully if queried, but
naive code that assumes success will crash or behave oddly). This is one of the most common
compatibility bugs in Vita homebrew — see [Input devices](06-input-devices.md).

## Storage tiers at a glance

The Vita exposes several distinct mount points (`ux0:`, `ur0:`, `uma0:`, `imc0:`, `xmc0:`, `pd0:`,
`host0:`, ...), not a single unified filesystem — see
[Storage & filesystem](05-storage-filesystem.md) for the full breakdown and what each is actually
for.

## Why this matters for homebrew

Nearly everything unusual about writing Vita code compared to, say, a modern mobile/desktop target
traces back to this being 2011-era mobile hardware with a memory architecture split into physically
separate pools (system RAM vs VRAM vs a small physically-contiguous pool used by hardware
video/audio decode) rather than a single unified heap the OS pages transparently. Expect to think
about *which* pool of memory something needs to live in, not just how much memory it needs — see
[Memory architecture](04-memory-architecture.md).
