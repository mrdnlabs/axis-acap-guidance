# Record Now, Analyze Later: On-Camera Batch Processing

> **Mostly empirical.** The pattern and numbers below come from two proof-of-concept ACAPs (a head
> counter and a pose estimator) on an AXIS P3288-LVE (ARTPEC-9, AXIS OS 13.1.19) and an AXIS
> P3408-VE (ARTPEC-9, AXIS OS 12.11). The official facts behind it (VDO encodes but does not decode;
> the bundled libraries) are linked under [References](#references). Verify on your device.

## The idea

Live analytics must keep up with the frame rate, which caps model size and input resolution. Many
scenes are empty most of the time, though. An ACAP can **record while something interesting is in the
scene, then re-analyze the recording once the scene is empty**: every frame, a bigger model, a higher
input resolution, and algorithms that need the *future* (bidirectional smoothing, whole-clip tracking).
Processing may then take longer than real time; the idle time pays for it.

## Building blocks

| Need | What works | Notes |
|---|---|---|
| Trigger | a small live model (or an AXEvent from a bundled analytic) with a presence state machine | enter after N consecutive frames with a confident detection, leave after M seconds without |
| Recording | a VDO stream with `format = VDO_FORMAT_JPEG` (Motion-JPEG), kept running | the camera's hardware encoder does the work |
| Pre-roll | a RAM ring of the last few seconds of JPEG frames | flushed into the recording when the trigger fires, so the entry is not lost |
| Storage | the app's `localdata/` directory (internal flash) | or an SD card through AXStorage |
| Decode | libjpeg-turbo, bundled in the package (`lib/` + `-Wl,-rpath,'$ORIGIN/lib'`) | `tjDecompressToYUVPlanes` → rebuild NV12 → the same larod preprocessing as live frames |
| Inference | a second larod model on the same DLPU | it can stay loaded; loading it costs a DLPU compile on first use |

### Why Motion-JPEG and not H.264

ACAP's VDO API encodes video but has no decode path, so the app must decode its own recordings.
Every JPEG frame stands alone, which makes random access, resume and per-frame review trivial, and
libjpeg-turbo is small. H.264/H.265 would be 5–10× smaller but needs a cross-compiled software decoder
(e.g. FFmpeg's libavcodec). **Inference:** MJPEG is the simpler choice for clips of minutes; use H.264
plus a bundled decoder only for long recordings.

## Measured (empirical)

| | P3288-LVE, AXIS OS 13.1 | P3408-VE, AXIS OS 12.11 |
|---|---|---|
| JPEG 1280×720 @ 10 fps from VDO | ~75 KB per frame (≈ 0.75 MB/s), no dropped frames | — |
| JPEG 1920×1080 / 3840×2160 | — | 0.22 MB / 0.8 MB per frame; the encoder delivered only 1 fps / 0.3 fps while shared with other apps |
| libjpeg-turbo decode on the CPU | 23 ms per 1280×720 frame | 77 ms (1080p), 251 ms (4K) |
| Free flash for recordings | ~1.4 GB | ~1.2 GB |

- **The live NV12 stream and the JPEG stream carry the same VDO timestamps** for the same capture
  (median offset 0 µs over a 103 s clip), so live results can be matched to recorded frames by timestamp
  exactly, e.g. to compare live and batch output frame by frame.
- **Keep the live model running while batch processing**, at a reduced rate (e.g. 5 fps instead of 10),
  so a new arrival is still detected; pause the batch job when it is, and resume it from a saved frame
  index afterwards. Both models share the DLPU through larod without problems.
- **Split long visits**: cap a clip's length (e.g. 120 s) and immediately start the next one, so storage
  limits and batch jobs stay bounded.
- RAM was too tight for recordings on these 2 GB-class devices (~300–370 MB available with other
  analytics running); write to flash and keep a free-space floor (e.g. stop at 150 MB left).

## Design for testability

Expose the state machine and every job over HTTP (`status.cgi`: present/empty, recording id and frame
count, batch progress, ms per frame, "seconds of processing per second of video"), add a **manual
recording trigger** so the batch path can be tested with nobody in the scene, and store per-frame
results as JSON lines next to the recording so a page or script can compare live and batch output.

---

## References

- [VDO API](https://developer.axis.com/acap/api/native-sdk-api/) — video streams in YUV/NV12, H.264/H.265 and JPEG; there is no decode API.
- [Supported APIs — ACAP Native SDK](https://developer.axis.com/acap/reference/supported-apis/) — VDO, Larod, AXStorage. libjpeg-turbo isn't on that list. **Observed:** these projects bundle `libturbojpeg.so.0` in the package rather than relying on the device's copy.
- [acap-native-sdk-examples: vdostream, vdo-larod, recording-playback](https://github.com/AxisCommunications/acap-native-sdk-examples) — stream setup, VDO → larod preprocessing, and (for the camera's own edge recordings) searching and retrieving recording files.
- [libjpeg-turbo TurboJPEG API](https://libjpeg-turbo.org/Documentation/Documentation) — `tjDecompressToYUVPlanes`.
