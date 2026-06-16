# Axis C6110-E Hardware Notes

> **Most of this file is empirical, device-specific behavior** observed on C6110-E firmware — the `dspcontrol` daemon, PipeWire node names, the ALSA re-mute race, the 48 kHz F32P negotiation, the `transmit.cgi` 503 phrase, and the paging-console-actions endpoint are **not** in Axis documentation. Verify on your firmware. Officially-sourced claims are linked under [References](#references).

## dspcontrol and the Speaker Guard Problem

The C6110 runs a system daemon called `dspcontrol` (PID ~637) that enforces hardware audio policy. It holds `/dev/snd/controlC0` open with `epoll` and a `timerfd`.

**Key behavior:** When headphones are plugged into the 3.5mm jack, `dspcontrol` automatically mutes the built-in speaker via ALSA controls. Any VAPIX `setDevicesSettings` unmute call writes an ALSA control, which wakes `dspcontrol` via epoll, which immediately re-mutes the speaker — within milliseconds.

This makes a simple VAPIX polling approach (periodically re-unmuting the speaker) fundamentally unworkable, no matter how fast you poll.

**Workaround deployed (Approach B):** Create a PipeWire loopback — capture from `AudioDevice0Output0` (headphone monitor) and play back to `AudioDevice0Output1` (speaker). This keeps the speaker's PCM stream active even while the hardware mute is engaged.

---

## PipeWire Node Names (C6110-E)

| Node Name | Description |
|-----------|-------------|
| `AudioDevice0Output0` | Headphone jack output |
| `AudioDevice0Output1` | Built-in speaker output |
| `AudioDevice0Outputs` | Combined output monitor (use this for capture to forward all audio) |

These are fixed hardware identifiers on the C6110-E.

---

## Audio Forwarding to C1110-E

- Capture from `AudioDevice0Outputs` at **48 kHz F32P** (the device negotiates this rate; you cannot request 8 kHz directly)
- Downsample 6:1 in software (48000 / 8000 = 6) for G.711 µ-law encoding
- POST encoded audio to `http://<C1110-IP>/axis-cgi/audio/transmit.cgi`
- Use Digest auth
- A `503 "Receiving audio from another client"` response was observed to mean another source is already feeding the device (only one transmit session at a time). Note: this exact reason phrase is **not** in the current VAPIX Audio API docs (the documented 503 reason is *"The maximum number of clients are already connected"*), so treat it as observed device behavior rather than a documented contract — verify on your firmware

---

## Paging Console Actions API

Actions for paging console buttons are managed via:
```
http://127.0.0.1/config/rest/paging-console-actions/v1/actions
```

- `GET` returns existing actions
- `POST` creates a new action
- Action `type: "httpRequest"` triggers an HTTP GET/POST when the button is pressed
- Check for duplicate `label` values before creating to avoid accumulating duplicates on each ACAP restart

This endpoint is **not** in the published VAPIX reference. It follows the [Device Configuration APIs](https://developer.axis.com/vapix/device-configuration/device-configuration-apis/) `/config/rest/<api>/v<n>` convention (DCA, AXIS OS 11.8+) but the specific `paging-console-actions` API is observed on-device only — verify on your firmware.

---

## VAPIX Authentication

Use Digest authentication for all local VAPIX calls. See [vapix-local-auth-from-acap.md](./vapix-local-auth-from-acap.md) for the D-Bus credential retrieval pattern.

---

## References

- [VAPIX — Audio API](https://developer.axis.com/vapix/audio-systems/audio-api/) — `/axis-cgi/audio/transmit.cgi` for sending audio to a device for playback. (The documented 503 reason is *"The maximum number of clients are already connected"*; the `"Receiving audio from another client"` phrase noted above is observed, not documented.)
- [VAPIX — Audio Device Control](https://developer.axis.com/vapix/audio-systems/audio-device-control/) — `audiodevicecontrol.cgi`; see [vapix-audio-device-control.md](./vapix-audio-device-control.md) for the mute/topology details.
- [VAPIX — Device Configuration APIs](https://developer.axis.com/vapix/device-configuration/device-configuration-apis/) — the `/config/rest/<api>/v<n>` REST convention used by the paging-console-actions endpoint.
- [AXIS C6110 product documentation](https://help.axis.com/en-us/axis-c6110) — confirms the 3.5 mm headphone connector, built-in speaker, and Actions configuration.

*The `dspcontrol` daemon, PipeWire node names, the ALSA re-mute race, and the 48 kHz F32P negotiation are device-internal behavior with no official Axis source — treat them as observed on the tested firmware.*
