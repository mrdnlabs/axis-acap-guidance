# DLPU Memory Contention: When a Bundled Analytic Starves Your Model

> **Empirical, observed behavior.** The failure signature below (an ACAP reporting `Status="Running"`
> while its model never loads, with OOM entries in syslog) was observed on an ARTPEC-9 dome with
> 2 GB RAM running AXIS OS 12.11, with AXIS Object Analytics active and a ~1440×1440 INT8 YOLOv5
> model. The mechanism generalizes, but the specific memory ceiling is per-device. Verify on your
> hardware. Officially-sourced claims are linked under [References](#references).

## The problem

Axis devices ship with bundled analytics (AXIS Object Analytics, Video Motion Detection, Image
Health Analytics, Live Privacy Shield). Several of them **also use the DLPU** and load their own
deep-learning models. On a memory-constrained device, a bundled analytic and your ACAP's model can
each load fine *alone* and fail *together*.

RAM is a **per-model SKU decision, not a property of the SoC** — two cameras with the same ARTPEC
generation can ship different amounts. Check the datasheet's `Memory` field (`<n> GB RAM, <n> GB
Flash`) before assuming headroom. Devices in the 2 GB class leave little room for a large model
alongside a bundled analytic.

## The failure signature (why it's deceptive)

The app **installs and starts successfully**. Nothing in the install path fails. Specifically:

```bash
# Says the app is fine:
curl --anyauth -u "root:<password>" "https://<device-ip>/axis-cgi/applications/list.cgi"
#   ... Name="<your-app>" Status="Running" ...
```

...while the app's own status endpoint reports the model never initialized. Syslog is where the
truth is:

```bash
curl --anyauth -u "root:<password>" "https://<device-ip>/axis-cgi/admin/systemlog.cgi"
```

Look for these three together:

| Log line | Meaning |
|---|---|
| `custodio: would have triggered OOM` | Axis's memory custodian tripped — **this is the smoking gun** |
| `larod[...]: Session N killed since client's (:1.xxx) connection has been lost` | your larod client died mid-load, repeatedly |
| `video-object-detection[...]: detector_larod.c: FE ON - load model` | a **bundled analytic** is loading its own model onto the DLPU |

A kernel task dump listing `larod` among processes, and swap being added at
`/mnt/flash/var/lib/larod-swap/swap`, corroborate memory pressure.

Note that `applications/list.cgi` may report `<Resource name="DeepLearningProcessor" used="No" />`
for **every** app even while one is actively using it — do not rely on that field to find the
contender. Use syslog.

## Diagnosis

1. List what's running — bundled analytics are `Bundled="Yes"`:
   ```bash
   curl --anyauth -u "root:<password>" "https://<device-ip>/axis-cgi/applications/list.cgi"
   ```
2. Grep syslog for `larod`, `OOM`, and `detector_larod`.
3. Check the device's RAM in its datasheet (`Memory` field).

## Fix

Stop the contending analytic, then restart your app:

```bash
curl --anyauth -u "root:<password>" \
  "https://<device-ip>/axis-cgi/applications/control.cgi?action=stop&package=objectanalytics"
curl --anyauth -u "root:<password>" \
  "https://<device-ip>/axis-cgi/applications/control.cgi?action=restart&package=<your-app>"
```

Reverse with `action=start`. If your product must coexist with a bundled analytic, the levers are:
**shrink the model input** (DLPU memory and time scale with input pixel count), pick a device with
more RAM, or accept that the analytic stays off.

## The reset trap

**A factory reset re-enables the bundled analytics.** A camera that hosted your app happily for
months can fail immediately after a reset — not because anything in your app changed, but because
AXIS Object Analytics is running again. If your app previously worked on the exact same hardware and
now can't load its model, check what else is running before you debug your own code.

The same reset also re-arms `AllowUnsigned=false` — see
[enable-unsigned-apps.md](./enable-unsigned-apps.md).

## Practical guidance

- **Gate your deploy on the app's own health endpoint, not on `Status="Running"`.** "Installed and
  started" is not "working." An install-only check reports success on a device where the model is
  dead.
- Treat available RAM as a **per-SKU** fact to look up, not a per-SoC assumption.
- On a constrained device, decide early whether the bundled analytics are part of the product; if
  they are, budget the model accordingly.

---

## References

- [VAPIX — Application API](https://developer.axis.com/vapix/applications/application-api/) —
  `list.cgi`, `control.cgi` (`action=start|stop|restart&package=<name>`).
- [Axis DLPU (developer docs)](https://developer.axis.com/computer-vision/computer-vision-on-device/axis-dlpu/)
  — the DLPU is a property of the SoC; devices with the same SoC "perform similarly," with
  performance differences possible from clock speed and **memory**.
- [larod — introduction for app developers](https://developer.axis.com/acap/api/src/api/larod/html/md__opt_builder-doc_larod_doc_introduction-for-app-developers.html)
  — larod is a shared service that arbitrates DLPU access between processes.
- [Optimization tips](https://developer.axis.com/computer-vision/computer-vision-on-device/optimization-tips/)
  — increasing model input size increases inference time (and memory).
- [AXIS Object Analytics](https://www.axis.com/products/axis-object-analytics) — bundled on most
  modern Axis cameras and enabled by default.
