# Networking

## WiFi only

The Vita's only built-in network interface is WiFi (802.11 b/g/n) — there's no cellular modem on
WiFi-only SKUs relevant to homebrew (3G-model Vitas exist but homebrew networking code universally
targets the WiFi path), and PSTV models with Ethernet ports route through the same underlying
network stack rather than a fundamentally different API surface.

## The sceNet stack

Networking goes through Sony's own `sceNet` family rather than a BSD-sockets-identical API, though
it's BSD-*socket-shaped* enough that porting simple socket code is usually straightforward:

- **`sceNet`** — core socket API (`sceNetSocket`, `sceNetConnect`, `sceNetSend`/`sceNetRecv`, etc.),
  its own `SceNetSockaddrIn` struct family (not literally `struct sockaddr_in`, so socket code needs
  a thin translation layer, which is exactly what libraries like `curl`'s Vita port already provide).
- **`sceNetCtl`** — connection state/info (is WiFi actually connected, current IP, signal info) —
  check this before assuming networking is available; a homebrew app launched with WiFi off or out
  of range should degrade gracefully rather than hang on socket calls that will simply never
  succeed.
- **`sceHttp`** / **`sceSsl`** — Sony's own HTTP(S) client stack, which some libraries (and some
  homebrew directly) use instead of a generic sockets-based HTTP implementation.

## curl on Vita

Most homebrew that needs to talk HTTP(S) to arbitrary servers (checking for updates, downloading
content, hitting a JSON API) links **libcurl**, cross-compiled for VitaSDK's target, rather than
hand-rolling HTTP over raw `sceNet` sockets or using `sceHttp` directly. This is the path of least
resistance for portable code, since it lets you write ordinary curl-based C code without threading
Vita-specific socket-struct translation through your own networking layer.

**A gotcha worth knowing**: VitaSDK's prebuilt `libcurl.a`/`libSDL2_mixer.a` and similar libraries
are frequently built with several **optional codec/compression backends as separate static
dependencies** that aren't linked automatically — if you see undefined-reference linker errors for
symbols with prefixes like `op_*` (opusfile), `xmp_*` (libxmp), `ZSTD_*`, or similar, the fix is
almost always adding the specific backend library (`opusfile`, `opus`, `xmp`, `zstd`, ...) to your
own `target_link_libraries`, not a sign that curl/SDL2_mixer itself is broken. See
[VitaSDK: project setup & build system](../02-vitasdk/02-project-setup-build-system.md).

## Practical takeaways

- Always check connectivity state before assuming network calls will succeed — a homebrew app that
  hangs indefinitely on a socket call because WiFi is off is a common, avoidable UX failure.
- Prefer curl (or another portable networking library already ported to VitaSDK) over hand-rolling
  raw `sceNet` socket code, unless you have a specific reason to talk to `sceNet`/`sceHttp`
  directly.
- Watch for missing optional-backend linker errors when pulling in networking-adjacent libraries —
  they're almost always a missing `target_link_libraries` entry, not a broken library build.
