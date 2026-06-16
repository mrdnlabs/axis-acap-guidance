# Axis C6110-E Hardware Notes

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
- A `503 "Receiving audio from another client"` response was observed to mean another source is already feeding the device (only one transmit session at a time). Note: this exact reason phrase is **not** in the current VAPIX docs for `transmit.cgi` (the documented 503 there is on `receive.cgi`), so treat it as observed device behavior rather than a documented contract — verify on your firmware

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

---

## VAPIX Authentication

Use Digest authentication for all local VAPIX calls. See [vapix-local-auth-from-acap.md](./vapix-local-auth-from-acap.md) for the D-Bus credential retrieval pattern.
