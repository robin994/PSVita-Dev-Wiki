# vitaGL: Textures

## Creating and uploading textures

The familiar shape mostly holds: `glGenTextures`, `glBindTexture(GL_TEXTURE_2D, id)`,
`glTexImage2D(target, level, internalFormat, width, height, border, format, type, data)`. For a lot
of ordinary UI/2D texture work (icons, static images decoded via a library like `stb_image` into a
plain RGBA buffer), this is exactly what it looks like on any other GL-flavored API, and code
written this way tends to port over from other platforms with minimal fuss.

## Getting at the underlying sceGxm texture

Because vitaGL sits on top of sceGxm (see [Overview](01-overview.md) and
[Hardware: GPU architecture](../01-hardware/03-gpu-architecture.md)), it exposes an escape hatch to
reach the real underlying `SceGxmTexture` object for a currently-bound texture:
**`vglGetGxmTexture(target)`**. This matters whenever something *outside* vitaGL's own draw path
needs to write directly into a texture's backing memory or otherwise manipulate it at the sceGxm
level — the canonical example being video playback: a decoded video frame from `SceAvPlayer` isn't
produced by uploading pixel data through `glTexImage2D` each frame (far too slow, and not how the
decoder hands you data anyway) — instead, code creates a small placeholder texture up front, grabs
its `SceGxmTexture*` via `vglGetGxmTexture`, and then calls `sceGxmTextureInitLinear` directly on it
each frame, pointing it at the decoder's own output buffer address — updating what the texture
*points at* rather than copying pixel data through it. This pattern (mix ordinary vitaGL calls for
setup, drop to the raw sceGxm object for a specific performance- or hardware-integration-sensitive
operation) is worth recognizing as a general escape-hatch technique, not something specific to video.

## Texture formats

vitaGL supports the standard RGBA/RGB-family formats you'd expect for ordinary image data
(`GL_RGBA`+`GL_UNSIGNED_BYTE` being the everyday default for decoded PNG/JPEG-style image data).
For video-frame textures specifically, the relevant sceGxm-level format is typically a planar
YUV format (`SCE_GXM_TEXTURE_FORMAT_YVU420P2_CSC1` being the one that shows up for
`SceAvPlayer`-sourced frames — a hardware-native planar chroma-subsampled layout with a built-in
colorspace-conversion tag, not something you'd construct by hand from an RGBA buffer) — set via the
same `sceGxmTextureInitLinear` call mentioned above rather than through `glTexImage2D`, since it's
not an ordinary vitaGL-managed texture upload path.

## Filtering

`sceGxmTextureSetMinFilter`/`sceGxmTextureSetMagFilter` (again, operating on the raw
`SceGxmTexture*`, for cases where you're already down at that level — e.g. right after the
`sceGxmTextureInitLinear` call above) or the more familiar `glTexParameteri` path for
ordinarily-managed textures — `SCE_GXM_TEXTURE_FILTER_LINEAR`/`SCE_GXM_TEXTURE_FILTER_POINT` at the
sceGxm level map onto the linear/nearest concepts you'd expect.

## Where texture memory actually lives

Every texture's backing storage is drawn from one of vitaGL's memory pools (`VGL_MEM_RAM` by
default for ordinary `glTexImage2D`-created textures) — see
[Memory pools deep dive](06-memory-pools-deep-dive.md) for when and why you'd want a texture routed
somewhere other than the default, and [Textures created for hardware-decoded video specifically] —
see [Hardware: multimedia hardware](../01-hardware/07-multimedia-hardware.md) — for a concrete case
where the *consumer* of the texture (not vitaGL itself) dictates which pool its backing memory has
to come from.

## Practical guidance

- For ordinary static image textures (icons, UI assets), the standard
  `glGenTextures`/`glTexImage2D` path is right — no need to reach for `vglGetGxmTexture` unless you
  have a specific reason to.
- Reach for `vglGetGxmTexture` + direct sceGxm calls specifically when something outside vitaGL's own
  upload path needs to write a texture's contents (hardware video decode being the primary real-
  world case), not as a general "more advanced/better" alternative to normal texture creation.
- Be deliberate about which memory pool a texture's storage comes from when the texture's *consumer*
  has requirements beyond "just render it" (video decode integration being the clearest example).
