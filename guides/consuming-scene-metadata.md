# Consuming AXIS Scene Metadata from an ACAP

How to read per-object detections, classes and tracks on-device — and the behaviors that will
surprise you once you do. This is the "consume, don't detect" path: the camera already detects,
classifies and tracks, so an application that only needs *what is where* should subscribe rather
than run its own model. No DLPU is involved, so it coexists with AXIS Object Analytics.

Complements [aoa-api-patterns.md](./aoa-api-patterns.md), which covers AOA's own
`control.cgi` counting API. This guide is about the raw object stream underneath.

> **Empirical unless cited.** Behaviors below were observed on AXIS Q3538-SLVE, AXIS OS 12.11.77,
> AOA 1.26.205, over a ~7-hour continuous sample (30,508 frames / 26,057 detections / 385 completed
> tracks). Schema and topic names are documented — see References.

---

## 1. Use the Device Data Hub, not the Message Broker

Two APIs can deliver this data. Pick the right one:

| | Message Broker (`libmdb`) | **Device Data Hub** (`device-data-hub-client-c`) |
|---|---|---|
| Topic | `com.axis.analytics_scene_description.v0.beta` | `com.axis.scene.frame.v1` |
| Status | Axis marks the topic **deprecated/superseded**; the API is **removed in AXIS OS 13.0** | current |
| Introduced | Native SDK 1.13 (AXIS OS 11.9) | Native SDK 12.10 |

New work should use the Device Data Hub. Building on `mdb` ships something that breaks at the next
major AXIS OS.

Manifest resource (note the key was renamed — see
[acap-manifest-gotchas.md](./acap-manifest-gotchas.md)):

```json
"resources": {
    "deviceDataHub": { "enabled": true }
}
```

```make
PKGS = device-data-hub-client-c
```

### Topics actually offered on a device

`dh_client_get_topic_list()` is the reliable way to find out what a given device publishes, rather
than assuming. On the test device it returned 16 topics, including:

```
com.axis.scene.frame.v1                 <- frame-by-frame detections (Fusion Tracker)
com.axis.scene.object_track.v1           <- one consolidated record per object, at track end
com.axis.scene.object_snapshot.v1        <- cropped object images (opt-in; see Privacy below)
com.axis.device.cpu-utilization.v1       <- useful for proving you added no load
com.axis.device.memory-utilization.v1
mdb.com.axis.analytics_scene_description.v0.beta
mdb.com.axis.consolidated_track.v1.beta
```

**The legacy Message Broker topics are bridged into the Data Hub under an `mdb.` prefix.** An
exact-string check for `com.axis.analytics_scene_description.v0.beta` reports *absent* while the
data is in fact available as `mdb.com.axis.analytics_scene_description.v0.beta`. Worth knowing if
you need a fallback, and worth not being fooled by.

The two `com.axis.device.*-utilization` topics are a better basis for "my app added no measurable
load" than sampling from outside the device.

---

## 2. Payload shape

```json
{
  "channel_id": 1,
  "timestamp": "2026-07-29T11:55:30.369150Z",
  "detections": [
    {
      "object_track_id": "8e653185-8e54-4dca-8af1-9f7202aca120",
      "bounding_box": { "left": 0.4499, "top": 0.2365, "right": 0.7473, "bottom": 0.9853 },
      "class": { "type": "Human", "score": 0.7, "upper_clothing_colors": [ ... ] }
    }
  ],
  "track_events": [ { "type": "TrackEnded", "object_track_id": "..." } ]
}
```

Things that catch people out:

- **`detections` and `track_events` are absent, not empty**, when there is nothing to report. Treat
  them as optional. (With jansson, `json_array_foreach` over a NULL array is a safe no-op.)
- **`object_track_id` is a UUID string, not an integer.** A record type using `int` will not survive
  contact with real data.
- **Attributes live inside `class`**, alongside `type` and `score` — clothing colours, license plate
  details and so on. Read `class.type` / `class.score` and ignore the rest unless you need it.

### Coordinate frame

`bounding_box` is normalized **0–1**, origin **top-left**, X right, **Y increasing downward** — so
`bottom > top` and `right > left` on every well-formed box. Zone polygons expressed in the same
normalized frame need no transform at all.

A cheap self-check worth logging on the first detection, because getting this backwards silently
inverts every geometric test:

```c
syslog(LOG_INFO, "sanity bottom>top=%s right>left=%s",
       (bottom > top) ? "yes" : "NO", (right > left) ? "yes" : "NO");
```

---

## 3. Classification types, and two traps

Classification `type` is a **flat string**, not a nested parent/child. The full set:

`Human`, `Head`, `Vehicle`, `Car`, `Truck`, `Bus`, `Bike`, `VehicleOther`, `LicensePlate`

`Vehicle` is the generic fallback when a sub-type cannot be determined. `Bike` is the
motorcycle/bicycle bucket. Each classification carries a `score` in `[0,1]`.

### Trap 1: `Head` and `LicensePlate` are separate tracks

They are attribute detections of a parent object, but they arrive as **their own detection with
their own `object_track_id`**. One person therefore produces **two** tracks — a `Human` and a
`Head`.

Anything that counts objects, or times them, must exclude these or it will count and time the same
physical object twice. Over the sample: 3,078 `Human` detections and 699 `Head` detections, all from
the same handful of people.

### Trap 2: most detections have no class at all

**85% of detections carried no `class` object whatsoever** (22,280 of 26,057). A short sample is
badly misleading here — the first few minutes of the same scene suggested ~10%.

Two consequences:

- An unclassified detection is the **normal** case, not an edge case. Logic that discards
  unclassified detections discards most of the stream.
- A high minimum-score filter throws away most of what the camera reports. Tune it against a long
  sample, not a demo.

If you need classification before acting, record the object's position and first-seen time anyway,
and backdate once a class eventually arrives — otherwise you lose the early part of every object's
life.

---

## 4. Track continuity: absence is not departure

This is the single most important behavioral finding, and it is easy to get wrong.

**The tracker reuses one `object_track_id` across long absences.** Measured gaps of **41.9 s** and
**49.1 s** during which the object appeared in no frame at all, then resumed under the *same* id —
against a normal inter-frame interval of 0.1 s.

So an object missing from `detections` has **not** necessarily gone. If your application treats
absence as departure with a short timeout (5 s is a tempting default), it will end a track the
camera still considers continuous.

A rule that matches the source's own semantics:

- **Missing from a frame, no `TrackEnded`** → the tracker still owns it. Keep any timer running;
  start no timeout.
- **`TrackEnded` received** → *now* start whatever grace period you allow for re-acquisition.
- Leaving a region of interest is a separate question from being absent from the frame, and should
  be handled separately.

### `Rename` exists but was never emitted

`track_events` defines a `Rename` event (`from_id` → `to_id`) that signals two track ids are the
same object — the documented re-identification result.

**Across 385 completed tracks it fired zero times.** Re-association showed up purely as a reused
`object_track_id`. Handle `Rename` when it appears, but nothing may *depend* on it.

### Short spurious tracks are normal

Tracks lasting 0.1–0.5 s with no class and near-zero movement appear regularly. Any per-object
event needs a debounce, or every flicker becomes an event.

---

## 5. The stream is event-paced, not video-paced

Frame rate on the same fixed scene:

| Scene | Rate |
|---|---|
| Nothing happening | **~0.5 fps** |
| One person moving | ~3.8 fps |
| 7-hour average | ~1.25 fps |

Good news for CPU. But it breaks a specific, plausible piece of code: **detecting a clock step by
comparing consecutive metadata timestamps.**

A quiet scene and an NTP correction look identical if frame timestamps are all you compare — so
every quiet period reads as a clock jump. Carry a monotonic reading alongside each frame and compare
how much *each* clock advanced; only divergence between them is a real step:

```c
int64_t frame_delta = frame_us - prev_frame_us;
int64_t mono_delta  = monotonic_us - prev_mono_us;      /* g_get_monotonic_time() */
if (llabs(frame_delta - mono_delta) > limit) { /* genuine clock step */ }
```

Timestamps are UTC from the device's NTP-synced clock, ISO-8601 with microseconds.

---

## 6. Privacy

`com.axis.scene.object_snapshot.v1` carries **cropped JPEG images of detected objects**, base64 in
the payload. It requires explicit enablement, and `object_track.v1` records can also carry an
`image` member.

If your application does not need imagery, leave snapshots disabled and strip any `image.data`
before logging a payload — otherwise a debug log quietly becomes a store of pictures of people.
Enabling it changes the privacy posture of the application and should be a deliberate, reviewed
decision, not a side effect of turning on verbose logging.

---

## References

- [AXIS Scene Metadata](https://developer.axis.com/analytics/axis-scene-metadata/) — overview.
- [Data sources](https://developer.axis.com/analytics/axis-scene-metadata/reference/data-sources/) —
  topic names, availability, and the deprecation of `com.axis.analytics_scene_description.v0.beta`
  in favour of `com.axis.scene.frame.v1` (AXIS OS 12.8+).
- [Analytics Data Format — Frame](https://developer.axis.com/analytics/axis-scene-metadata/reference/data-formats/analytics-data-format/frame/) —
  `channel_id`, `timestamp`, `detections`, `track_events` (`TrackEnded`, `Rename`).
- [Analytics Data Format — basic types](https://developer.axis.com/analytics/axis-scene-metadata/reference/data-formats/analytics-data-format/basic-types/) —
  `ObjectDetection`, `BoundingBox` normalized 0–1.
- [Classification types](https://developer.axis.com/analytics/axis-scene-metadata/reference/data-formats/analytics-data-format/classification-types/) —
  the full `type` enum and per-class attributes.
- [Re-identification](https://developer.axis.com/analytics/axis-scene-metadata/reference/concepts/re-identification/) —
  how `Rename` is meant to expose track consolidation.
- [Device Data Hub API](https://developer.axis.com/acap/api/native-sdk-api/#device-data-hub-api) —
  and the note that the Message Broker API is removed in AXIS OS 13.0.
- [`device-data-hub/consume-scene-metadata` example](https://github.com/AxisCommunications/acap-native-sdk-examples/tree/main/device-data-hub/consume-scene-metadata) —
  official subscription example. Check which SDK it targets before copying; see
  [sdk-firmware-version-matching.md](./sdk-firmware-version-matching.md).
