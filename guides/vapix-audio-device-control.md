# VAPIX: audiodevicecontrol.cgi — Muting Audio Outputs

## The Problem

Calling `setDevicesSettings` with a top-level `mute` field on an output **does nothing**. It returns HTTP 200 and no error, but the mute state does not change.

**Broken (silently ignored):**
```json
{
  "apiVersion": "1.0",
  "method": "setDevicesSettings",
  "params": {
    "devices": [{"id": "0", "outputs": [{"id": "1", "mute": true}]}]
  }
}
```

## The Fix

You must use the **full nested path** down to the channel level:

```
devices[].outputs[].connectionTypes[].signalingTypes[].channels[].mute
```

**Working:**
```json
{
  "apiVersion": "1.0",
  "method": "setDevicesSettings",
  "params": {
    "devices": [{
      "id": "0",
      "outputs": [{
        "id": "1",
        "connectionTypes": [{
          "id": "internal",
          "signalingTypes": [{
            "id": "unbalanced",
            "channels": [{"id": "0", "mute": true}]
          }]
        }]
      }]
    }]
  }
}
```

## How to Discover the Correct Topology

Call `getDevicesSettings` first and inspect the response to find the correct `connectionType`, `signalingType`, and `channel` IDs for each output on your device.

## Device Topologies (empirically confirmed)

The IDs below were read back from `getDevicesSettings` on the listed devices. Axis does **not** publish per-model channel/connectionType/signalingType IDs, so there is no official source for these tables — always re-run `getDevicesSettings` to confirm on your device and firmware.

### Axis C6110-E

| Output | `out_id` | `connType`  | `sigType`    | `channel_ids` |
|--------|----------|-------------|--------------|---------------|
| OUT0 — Headphone jack | `"0"` | `"headphone"` | `"unbalanced"` | `["0", "1"]` |
| OUT1 — Built-in speaker | `"1"` | `"internal"` | `"unbalanced"` | `["0"]` |

### Axis C1110-E

| Output | `out_id` | `connType`  | `sigType`    | `channel_ids` |
|--------|----------|-------------|--------------|---------------|
| OUT0 — Speaker | `"0"` | `"internal"` | `"unbalanced"` | `["0"]` |

All devices use `dev_id = "0"`.

---

## References

- [VAPIX — Audio Device Control](https://developer.axis.com/vapix/audio-systems/audio-device-control/) — `audiodevicecontrol.cgi`, the `getDevicesSettings` / `setDevicesSettings` methods, and the data model in which `mute` is defined only at the **channel** level (`devices[].outputs[].connectionTypes[].signalingTypes[].channels[].mute`). There is no `outputs[].mute` field — which is exactly why a top-level mute is accepted and silently ignored.
