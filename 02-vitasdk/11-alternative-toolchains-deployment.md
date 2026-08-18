# Alternative toolchains & deployment tools

[Overview & toolchain](01-overview-toolchain.md) covers VitaSDK and `vdpm` as this wiki's baseline
toolchain assumption. This page fills in `vdpm` itself in more depth, plus the adjacent
tools/ecosystems a project might reach for instead of or alongside it.

## `vdpm` — the VitaSDK package manager, in depth

**[vdpm](https://github.com/vitasdk/vdpm)** is a rootless Pacman client scoped entirely inside
`$VITASDK` — it never touches the host system's own package database or `/usr`. Packages come from
signed channel manifests: the manifest signature is checked against
`$VITASDK/share/vdpm/channel-public-key.pem`, and the Pacman databases themselves are SHA-256
verified before use. On Windows it ships as a portable native `vdpm.exe` that calls Pacman directly
without needing Bash; on Linux/macOS it's a full package-manager build against pinned Pacman/libalpm
sources.

```sh
git clone https://github.com/vitasdk/vdpm
cd vdpm
./bootstrap-vitasdk.sh

vdpm install zlib sdl2      # install packages
vdpm upgrade                # update the whole toolchain
vdpm refresh nightly        # switch release channel
```

**Migration gotcha**: bootstrap can't describe file ownership for an existing legacy (pre-vdpm) SDK
install to Pacman, so it replaces rather than converts one — plan for a clean install rather than an
in-place upgrade if migrating an old toolchain setup. Ongoing `vdpm upgrade` from a vdpm-bootstrapped
install works fine; it's only the first migration that's disruptive. Trust model is intentionally
GitHub/HTTPS-delivery-dependent for the initial bootstrap (not pinned-digest) — a deliberate design
choice per the project, not an oversight.

## VitaDeploy — device-resident, offline homebrew toolbox

**[VitaDeploy](https://github.com/SKGleba/VitaDeploy)** runs entirely on the device itself, bundling
several separately-maintained tools into one interface rather than requiring a PC connection at all.
It **includes VitaShell as its file manager component** rather than replacing it — the distinction
from `vitacompanion` (see [Debugging & tooling](07-debugging-tooling.md)) is that VitaDeploy targets
offline, device-resident setup/maintenance tasks (firmware install across 3.60/3.65/3.68 with
downgrade/upgrade, `sd2vita` mounting, internal memory unlocking, plugin preset downloads, battery
reset, kiosk-mode toggling), while `vitacompanion` targets the PC-connected build/deploy/relaunch
loop this wiki's own workflow depends on day to day. They solve adjacent but different problems —
VitaDeploy is what you'd hand a non-developer setting up a console for the first time; the
`vitacompanion`/FTP/`nc` loop is what you'd use while actively iterating on a build.

## VDSuite — a third SDK/toolchain lineage

Several other tools researched for this page ([QuickMenuReborn](13-system-integration-libraries.md),
[VitaMonoLoader](12-alternative-runtimes-openal.md)) mention supporting "VDSuite" or "VDS" alongside
VitaSDK — this is a **separate, third toolchain/devkit lineage** distinct from both open-source
VitaSDK and Sony's own leaked official SDK, used by a subset of the homebrew community as an
alternative build target. The list entry linking to it
(`forum.devchroma.nl/index.php?topic=332.0`, described only as "Additional features for the
PlayStation Vita SDK") **returned HTTP 404 at time of writing** — the thread appears to no longer
exist at that URL. If a project you're evaluating targets VDSuite specifically rather than VitaSDK,
expect to need its own toolchain setup separate from everything else in this wiki's VitaSDK-centric
build instructions, and verify the current canonical source for it before relying on that dead link.

## PSM Reborn — PlayStation Mobile archive, historical only

**[PSM Reborn](http://psmreborn.com/devtools.php)** is a preservation/archive site for PlayStation
Mobile (PSM) development resources — PSM was a distinct Sony platform for independent developers
that predates and is unrelated to the VitaSDK homebrew ecosystem this wiki otherwise covers, and is
no longer officially supported. The site organizes archived tools into four categories: PSM Unity,
core PSM resources, PSM Android, and unofficial community tools. Relevant only if you're
specifically dealing with legacy PSM content (e.g. [CXML-Decompiler](10-reverse-engineering-debugging.md)-adjacent
work); not part of the modern VitaSDK homebrew toolchain in any way.
