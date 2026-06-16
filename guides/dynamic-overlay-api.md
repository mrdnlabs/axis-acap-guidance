# Dynamic Text Overlay API

How to update dynamic text overlays on Axis cameras from an ACAP or script.

---

## Overview

Axis cameras support dynamic text slots (`#D1` through `#D16`) that can be embedded in video stream overlays. These slots can be updated programmatically via VAPIX, allowing ACAPs to display live data (counts, status, labels) directly on the video stream.

---

## Prerequisites

The operator must add a dynamic text token (e.g., `#D1`) to the camera's overlay text configuration. This is done through the device web UI under **Video > Overlays**. The ACAP can then update the value of that slot, but cannot create the overlay itself.

---

## API

### Update a text slot

```
GET /axis-cgi/dynamicoverlay.cgi?action=settext&text_index=<n>&text=<value>
```

| Parameter | Description |
|-----------|-------------|
| `text_index` | Slot number: `1` through `16` |
| `text` | The text to display. URL-encode special characters. |

### Slot-to-token mapping

| Token in overlay | `text_index` value |
|------------------|--------------------|
| `#D1` | `1` |
| `#D2` | `2` |
| ... | ... |
| `#D12` | `12` |
| `#D16` | `16` |

### Example

To update slot `#D12` with the text "Object count 42":

```bash
curl --anyauth -u root:<password> \
  "http://<camera-ip>/axis-cgi/dynamicoverlay.cgi?action=settext&text_index=12&text=Object%20count%2042"
```

From within an ACAP, use the loopback address with D-Bus credentials:

```c
snprintf(url, sizeof(url),
    "http://127.0.0.1/axis-cgi/dynamicoverlay.cgi"
    "?action=settext&text_index=%d&text=%s",
    slot_index, url_encoded_text);
```

---

## Gotchas

- **Slots are `#D1` through `#D16`, not `#D0`.** There is no slot zero.
- **The wrong CGI path is easy to guess.** The path is `dynamicoverlay.cgi`, not `videooutput/setdynamicoverlays.cgi` or other variations. Verify on your target device.
- **The operator must add the token to the overlay.** The ACAP can only set the text value — it cannot create the overlay configuration. If `#D12` is not in the overlay text, updating `text_index=12` has no visible effect.
- **URL-encode the text value.** Spaces, special characters, and non-ASCII text must be properly encoded.
- **Dynamic text vs. custom overlay.** Dynamic text depends on the operator's OSD configuration. For a fully ACAP-controlled overlay (independent of OSD settings), use the `axoverlay` API instead. Consider supporting both modes.

---

## When to Use Each Approach

| Approach | Best for | Depends on operator config? |
|----------|----------|----------------------------|
| Dynamic text (`dynamicoverlay.cgi`) | Simple status text, low-friction integration | Yes — operator must add `#D<n>` token |
| Custom overlay (`axoverlay` API) | Rich graphics, large text, full ACAP control | No — ACAP manages its own overlay |
| Both | Maximum flexibility for different deployments | Partially |
