# ACAP Manifest Gotchas

## No "Open" button on the Apps page? You have no settingPage

The Apps page shows no **Open** button for your application unless the manifest declares a settings
page. There is no warning — the app installs and runs, and the button is simply absent.

```json
"configuration": {
    "settingPage": "index.html"
}
```

Two ways to get this wrong:

- **Omitting `configuration.settingPage` entirely.** The common case, and the one that sends you
  looking for a missing feature rather than a missing manifest key.
- **Spelling it `"settingsPage"`.** The key is `"settingPage"` — **singular**. The plural silently
  does nothing.

The HTML file goes in `app/html/` and is served at `/local/<appName>/`. `acap-build` packages the
whole `html/` directory; confirm before deploying:

```bash
tar tzf myapp.eap | grep html/
```

Once installed, `list.cgi` reports the resolved page — and that attribute is what renders the button:

```bash
curl --anyauth -u "root:<password>" "http://<device-ip>/axis-cgi/applications/list.cgi"
# ... ConfigurationPage="local/myapp/index.html" ...
```

If `ConfigurationPage` is absent from that output, the manifest is the problem, not the web server.

---

## schemaVersion must match SDK

The `schemaVersion` in `manifest.json` must be a version supported by the SDK you're building with.

- SDK `axisecp/acap-native-sdk:12.4.0-aarch64` supports up to **`1.7.4`**
- Specifying a schema the SDK doesn't ship causes a build failure, e.g. `ValueError: No schema matching schema version '1.9.0'`

The mapping is SDK-specific and moves with each release (e.g. `1.9.0` was introduced in SDK 12.8, and `2.0.0` — a breaking restructure — in SDK 12.10). Don't hardcode a number from memory; check available schemas inside the container — note there is a `v2/` directory once you are on schema 2.x:
```
/opt/axis/acapsdk/axis-acap-manifest-tools/schema/schemas/v1/
/opt/axis/acapsdk/axis-acap-manifest-tools/schema/schemas/v2/
```

```bash
docker run --rm --entrypoint /bin/bash axisecp/acap-native-sdk:<ver>-<arch>-ubuntu24.04 \
  -c "ls /opt/axis/acapsdk/axis-acap-manifest-tools/schema/schemas/*/"
```

Observed: **SDK 12.10 ships `v2.0.0` as its newest schema; SDK 12.11 ships `2.0.0`, `2.1.0` and
`2.2.0`.** So a manifest copied from a current Axis example (which will use the newest schema) is
rejected by an SDK one release behind — see
[sdk-firmware-version-matching.md](./sdk-firmware-version-matching.md).

---

## Schema 2.x: required setup fields, and `compatibleOsVersions` now gates installation

Schema 2.x requires more in `acapPackageConf.setup` than v1 did. All of these are mandatory:

`appName`, `architecture`, `compatibleOsVersions`, `runMode`, `vendor`, `vendorId`, `version`

Two are easy to trip over:

- **`vendorId` must match `^[A-Fa-f0-9]{10}$`** — exactly ten hex digits. It is an Axis-assigned
  identifier, so there is no correct value to invent; a format-valid placeholder will build and
  install but must be replaced before release.
- **`compatibleOsVersions` items require *both* `min` and `max`.** Supplying only one fails
  validation with a schema error naming `required: ['min', 'max']`.

```json
"compatibleOsVersions": [ { "min": "12.11.72", "max": "13" } ]
```

**The bigger change: from AXIS OS 12.10 this field prevents installation.** Per the schema
description — *"Starting with AXIS OS 12.10, the application will be prevented from being installed
on devices with an AXIS OS version not within any of these ranges."* It is no longer documentation;
get the range wrong and the package will not install on the device you meant it for.

Related: each SDK has its own minimum AXIS OS, which is narrower than the major.minor. Setting `min`
below it is a build **warning**, not an error, and is worth heeding:

```
* Warning: compatibleOsVersions entry 0: 'min' is set to '12.11', which is below
  the SDK's minimum AXIS OS version '12.11.72'
```

---

## Resource keys can be renamed between schema versions

`resources` keys are not stable across schema versions. Device Data Hub access is declared as:

| Manifest schema | Key |
|---|---|
| `2.0.0` (newest in SDK 12.10) | `deviceDataHub_beta2` |
| `2.2.0` (SDK 12.11) | `deviceDataHub` — `_beta2` still accepted |

```json
"resources": { "deviceDataHub": { "enabled": true } }
```

Using the wrong spelling fails validation with a message that dumps the whole `resources` sub-schema
and does not obviously say "you used the old name". If a resource key is rejected, list the keys the
schema you are building against actually accepts:

```bash
python3 -c "import json,sys; s=json.load(open(sys.argv[1])); \
print(list(s['properties']['resources']['properties']))" \
  application-manifest-schema-v2.2.0.json
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

When making VAPIX calls from within an ACAP to the same device, always use a loopback address — never the device's external LAN IP (it will break on any device with a different IP than the one you developed on). Use **`127.0.0.12`** when authenticating with D-Bus service-account credentials (that's the virtual host they're bound to — see [vapix-local-auth-from-acap.md](./vapix-local-auth-from-acap.md)); `127.0.0.1` is the plain loopback for calls that don't depend on those credentials.

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

---

## References

- [Manifest schema — field descriptions](https://developer.axis.com/acap/reference/manifest-schemas/manifest-v1/schema-field-descriptions-v1.10.0/) — `settingPage` ("must be located in directory 'html'"), `embeddedSdkVersion`, `runMode`, and the `reverseProxy`/`httpConfig` fields.
- [Manifest schemas — general info / version history](https://developer.axis.com/acap/reference/manifest-schemas/general-info/) — schema↔SDK mapping (schema 1.9.0 ← SDK 12.8; 2.0.0 ← SDK 12.10) and the note that schema 2.0.0 **removed** `embeddedSdkVersion`.
- [AXParameter — `ax_parameter.h` types](https://developer.axis.com/acap/api/native-sdk-api/) — the `Password` (masked, `*`), `Hidden` (hidden in UI only), and deprecated `WriteOnly` ("can neither be read by axparameter nor by param.cgi") parameter types.
- [ACAP 12.7 release notes](https://developer.axis.com/acap/release-notes/12.7/) — parameter changes recorded in the audit log, with `password` and `writeonly` values masked.
- [Manifest schema v2 — field descriptions](https://developer.axis.com/acap/reference/manifest-schemas/manifest-v2/schema-field-descriptions-v2.0.0/) — the schema 2.x `setup` requirements (`vendorId`, `architecture`, `compatibleOsVersions`), and the statement that from AXIS OS 12.10 an application is **prevented from being installed** on a device outside the declared `compatibleOsVersions` ranges.
- [VAPIX — Application API (`control.cgi`, `list.cgi`)](https://developer.axis.com/vapix/applications/application-api/) — `action=restart` / `start` / `stop` / `remove`, and the `ConfigurationPage` attribute in `list.cgi` output.
- [VAPIX — parameter management (`param.cgi`)](https://developer.axis.com/vapix/network-video/parameter-management/) — `action=list&group=…` to discover stored parameter names.

> **Note on `embeddedSdkVersion`:** Axis's field description frames it as the *minimum required SDK version the device must support* (a compatibility floor, set to `"3.0"`), rather than literally a "runtime/framework build number." The practical guidance above (use `"3.0"` on v1.x schemas, never `"4.0"`, omit on schema ≥ 2.0.0) is unaffected.
>
> Several items in this guide are **empirical/observed** and have no official source: the in-container schema path, the exact build-error and warning strings, which schema versions a given SDK ships (12.10 → `2.0.0`; 12.11 → `2.2.0`), the `deviceDataHub_beta2` → `deviceDataHub` key rename, `acap-build` merging fields from a stale `.eap`, the stale-`COPY` → `type="String"` symptom, the bash `!` history-expansion mangling, the `ChunkedEncodingError`-on-restart symptom, and the parameter-group capitalization rule. Treat these as field notes — the schema-2.x items were verified on **AXIS OS 12.11.77 with SDK 12.10.0 and 12.11.0**; the rest on AXIS OS 12.x.
