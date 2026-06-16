# ACAP Manifest Gotchas

## settingPage key is singular

The key to declare a web UI settings page is `"settingPage"` (singular), **not** `"settingsPage"`.

```json
"configuration": {
    "settingPage": "index.html"
}
```

The HTML file goes in `app/html/` and is served at `/local/<appName>/`.

---

## schemaVersion must match SDK

The `schemaVersion` in `manifest.json` must be a version supported by the SDK you're building with.

- SDK `axisecp/acap-native-sdk:12.4.0-aarch64` supports up to **`1.7.4`**
- Specifying a schema the SDK doesn't ship causes a build failure, e.g. `ValueError: No schema matching schema version '1.9.0'`

The mapping is SDK-specific and moves with each release (e.g. `1.9.0` was introduced in SDK 12.8, and `2.0.0` — a breaking restructure — in SDK 12.10). Don't hardcode a number from memory; check available schemas inside the container at:
```
/opt/axis/acapsdk/axis-acap-manifest-tools/schema/schemas/v1/
```

---

## Parameter group capitalization is unpredictable

ACAP parameters are stored internally as `root.<AppName>.*` where capitalization of `AppName` can differ from `appName` in the manifest. For example, an app with `"appName": "audio_control"` may store params under `root.Audio_control.*` (capital A).

This does **not** affect `ax_parameter_get()` — always use just the short key name (e.g. `"RemoteIP"`, not `"root.Audio_control.RemoteIP"`).

It **does** matter when calling `param.cgi` from external code or scripts — use the actual stored name, which you can discover with:
```
/axis-cgi/param.cgi?action=list&group=root.<AppName>
```

---

## ACAP parameters are read only at startup

`ax_parameter_get()` reads live from the param store, but if you cache values at startup (e.g. in a `load_params()` function), the ACAP must be restarted to pick up changes made via the settings UI or `param.cgi`.

Design accordingly: mode switches and other real-time controls should call `ax_parameter_get()` at the time they're needed, not rely on cached startup values.

---

## Local VAPIX calls must use 127.0.0.1

When making VAPIX calls from within an ACAP to the same device, always use `http://127.0.0.1/axis-cgi/`. Using the device's external LAN IP will break on any device with a different IP than the one you developed on.

---

## curl and special characters in passwords

When setting parameters with `!` (or other shell-special characters) in passwords via `curl` from the shell, the `!` gets mangled by bash history expansion even in non-interactive shells with some invocations.

Use Python `requests` instead for reliability:
```python
import requests
requests.get(
    'http://<device-ip>/axis-cgi/param.cgi',
    params={'action': 'update', 'root.audio_control.RemotePass': 'S3cret!Value'},
    auth=requests.auth.HTTPDigestAuth('root', '<password>')
)
```

---

## ACAP restart via control.cgi drops the connection

`/axis-cgi/applications/control.cgi?action=restart` causes the ACAP process to restart, which closes the HTTP connection mid-response. This results in a `ChunkedEncodingError` in Python requests or a partial/empty response in curl — this is **normal**, not an error. The restart happened.

If restart doesn't seem to take effect, re-deploy the `.eap` — uploading forces a stop + start cycle.

---

## embeddedSdkVersion is NOT the SDK version

`embeddedSdkVersion` in manifest.json refers to the **ACAP framework version** (the runtime on the device), not the Native SDK build toolchain version.

- Use `"3.0"` — all official v1.x-schema examples use this value
- The SDK version (e.g., 12.9.0) is only used in the Dockerfile `FROM` tag
- There is no `"4.0"` value — it is invalid and can cause the ACAP to fail to start
- **Schema 2.0.0 (SDK 12.10+) removed this field entirely.** If you target `schemaVersion` ≥ `2.0.0`, omit `embeddedSdkVersion` rather than setting it

---

## reverseProxy: see dedicated guide

The `reverseProxy` manifest feature has several non-obvious behaviors. See [acap-reverse-proxy-guide.md](./acap-reverse-proxy-guide.md) for the full guide. Key surprise: **Apache forwards the full URI unchanged** — it does NOT strip the path prefix.

---

## paramConfig: `hidden:string` is UI-only, NOT a security control

The official `axparameter` SDK example declares `"type": "hidden:string"` for a "backup value" field. It's easy to read that and assume `hidden:` is the way to keep credentials out of view. **It is not.**

| Type modifier | UI rendering | `param.cgi?action=list` | Audit log (12.7+) | `ax_parameter_get()` |
|---|---|---|---|---|
| `String` | plain text field | returns value | logs value | returns value |
| `hidden:string` | **not shown** in Settings dialog | returns value (cleartext!) | logs value | returns value |
| `password` or `password:maxlen=N` | masked input (`*`) | returns `******` | logs `******` | returns value |
| `writeonly:string` *(deprecated)* | masked input | returns nothing | logs nothing | **returns nothing** — unusable |

Use `password:maxlen=N` for any field that holds a credential. `hidden:` only hides the UI tile; the value is still readable via `param.cgi` to any caller with VAPIX list access. See [standards §4.3](../standards/acap-development-standards.md#43-credential-storage).

The runtime accepts `password` (no suffix) or `password:maxlen=N`. The schema validator (`application-manifest-schema-v1.x.json`) only requires `"type"` to be a string, so the type string is opaque to the validator and gets normalized by `acap-build`'s `manifest2packageconf` step into `param.conf`. If your `param.conf` inside the built `.eap` shows `type="String"` for a field you declared as `password`, your Docker `COPY` layer is stale (see [acap-wsl-build-pitfalls.md](./acap-wsl-build-pitfalls.md)). Verify with:

```bash
tar -xOzf myapp.eap param.conf | grep MyPasswordField
# Expect: MyPasswordField="" type="password:maxlen=N"
```
