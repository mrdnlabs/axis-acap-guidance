# axoverlay: why your overlay looks like it isn't working

A custom overlay drawn with the `axoverlay` API can be entirely correct and still appear to do
nothing. The verification path is the part that catches people, and there are a couple of runtime
side effects worth knowing about too.

For choosing between a custom overlay and dynamic OSD text in the first place, see
[dynamic-overlay-api.md](./dynamic-overlay-api.md). This guide is about making a custom overlay work
once you have chosen it.

> **Empirical unless cited.** Observed on AXIS Q3538-SLVE, AXIS OS 12.11.77, ACAP Native SDK 12.11.0,
> using axoverlay2 with Cairo.

---

## 1. The JPEG snapshot will not show your overlay

This is the big one, because it produces a convincing false negative.

`/axis-cgi/jpg/image.cgi` opens its **own** VDO stream. An overlay attached to some other stream is
simply not present in that image. Grabbing a snapshot is the obvious way to check an overlay, and it
will show you a clean, unmodified frame while the overlay works perfectly on every stream a client is
actually watching.

**What works instead** — hold a stream open, then pull a frame out of *that same stream*:

```bash
# hold a stream open; this is also what triggers overlay creation
curl -s --anyauth -u "root:<password>" \
  "http://<device-ip>/axis-cgi/mjpg/video.cgi?resolution=1280x720&fps=5" -o stream.mjpg &
sleep 10
```

Then extract a frame. MJPEG is a multipart stream of complete JPEGs, each `FFD8…FFD9`:

```python
data = open("stream.mjpg", "rb").read()
start = data.rfind(b"\xff\xd8\xff")
end   = data.find(b"\xff\xd9", start)
open("frame.jpg", "wb").write(data[start:end + 2])
```

## 2. No stream means no overlay — and `axo_start()` succeeding tells you nothing

Overlays attach **per video stream**. On an idle camera with nobody watching, there may be no stream
to attach to, so nothing is created and nothing is drawn. `axo_start()` will still return success.

Log the per-stream creation and check for it before concluding anything about your drawing code:

```
OVERLAY_STARTED                          <- says nothing about whether anything is drawn
OVERLAY_CREATED stream=13 size=1280x720  <- this is the line that matters
```

Streams come and go as clients connect and disconnect, so expect create/remove churn during normal
use, and handle removal.

Use the VDO stream filter so you are not rendering for streams that will never display an overlay:

```c
VdoMap* filter = vdo_map_new();
vdo_map_set_string(filter, "filter", "overlay");
vdo_stream_attach(event_stream, filter, &error);
```

## 3. `AXO_ERR_WAIT` and `AXO_ERR_NO_STREAM` are normal

`axo_get_buffer()` legitimately fails during ordinary camera operation. Treat these two codes as
"skip this frame", not as errors worth logging — otherwise a working overlay fills the log:

```c
axo_buffer* buffer = axo_get_buffer(overlay_id, NULL, &err);
if (!buffer) {
    axo_err_code code = axo_err_get_code(err);
    if (code == AXO_ERR_NO_STREAM || code == AXO_ERR_WAIT)
        goto skip_frame;            /* expected */
    /* anything else is a real error */
}
```

The same applies to `axo_create_overlay()` returning `AXO_ERR_NO_STREAM` — a stream can close between
the event telling you it exists and your attempt to attach to it.

## 4. Cairo text floods the log with fontconfig errors

Cairo's text API pulls in fontconfig, which has no writable cache directory under an ACAP's
restricted home. It then logs an error on **every text draw**:

```
Fontconfig error: No writable cache directories
	/var/cache/fontconfig
	/.cache/fontconfig
```

At a couple of redraws per second with a few text items each, this floods the device syslog and
buries anything useful. Point `XDG_CACHE_HOME` somewhere writable before the first Cairo text call,
which silences it and lets the font cache actually work:

```c
char* cwd   = g_get_current_dir();
char* cache = g_build_filename(cwd, "localdata", "cache", NULL);
if (g_mkdir_with_parents(cache, 0755) == 0)
    g_setenv("XDG_CACHE_HOME", cache, TRUE);
```

## 5. Check the buffer size before copying into it

If you render into a Cairo image surface and then copy into the overlay buffer, the copy length is
derived from your own idea of the geometry. Verify it against the buffer you were actually handed:

```c
size_t needed = (size_t)full_width * full_height * 4u;
if (axo_buffer_get_byte_size(buffer) < needed)
    return;                         /* skip the frame; never overrun */
memcpy(target, cairo_image_surface_get_data(surface), needed);
```

Two related size rules:

- Always size the overlay through `axo_get_aligned_size()`; computed dimensions are often not
  aligned to what the overlay system supports.
- Confirm Cairo did not add its own stride padding, or a straight `memcpy` will shear the image:
  `cairo_image_surface_get_stride(surface) == full_width * 4`.

## 6. Sizing when you draw normalized geometry

If you are drawing something expressed in normalized (0–1) coordinates — a region of interest, a
detection box, anything derived from
[scene metadata](./consuming-scene-metadata.md) — make the overlay the **full stream size**. A
smaller overlay places your geometry somewhere other than where it belongs.

For high-resolution streams that gets expensive, so use the built-in 2× upscale: draw at half
resolution and let the hardware scale it.

```c
bool upscale = ((unsigned long)stream_w * stream_h) > 4000000;   /* ~4 MP threshold */
unsigned draw_w = upscale ? stream_w / 2 : stream_w;
axo_props_set_upscale_x2(props, upscale);
```

Overlay rendering runs on GPU/CPU and uses no DLPU, so it does not contend with a bundled analytic
the way a second inference model would — see [dlpu-memory-contention.md](./dlpu-memory-contention.md).
It is not free, though: keep the redraw rate matched to how fast the content actually changes. Text
showing whole seconds does not need 30 fps.

---

## References

- [Axoverlay 2 API](https://developer.axis.com/acap/api/native-sdk-api/#axoverlay-2-api) — `axo_start`,
  `axo_create_overlay`, `axo_get_buffer`, `axo_submit_buffer`, `axo_get_aligned_size`,
  `axo_props_set_upscale_x2`, and the `axo_err_code` values including `AXO_ERR_WAIT` and
  `AXO_ERR_NO_STREAM`.
- [`axoverlay2` example (ACAP Native SDK)](https://github.com/AxisCommunications/acap-native-sdk-examples/tree/main/axoverlay2) —
  the VDO stream-event pattern for creating and removing overlays per stream.
- [Video Capture API (VDO)](https://developer.axis.com/acap/api/native-sdk-api/#video-capture-api) —
  stream 0 as the pseudo-stream that reports events about all other streams, and the stream filter map.
- [VAPIX — media streaming](https://developer.axis.com/vapix/network-video/media-streaming/) —
  `mjpg/video.cgi` and `jpg/image.cgi`, the two endpoints whose separate streams cause the false
  negative in §1.
- [Cairo](https://developer.axis.com/acap/api/native-sdk-api/#cairo) — the 2D rendering library used
  to draw into the overlay buffer.
