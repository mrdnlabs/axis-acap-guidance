# Signing an ACAP Application

From AXIS OS 12.0, devices install only **signed** ACAP applications by default (unsigned is a toggle — see [enable-unsigned-apps.md](./enable-unsigned-apps.md) — that Axis plans to remove around AXIS OS 13). Signing is the durable path. This guide covers how to get an `.eap` signed, including the route for developers who are **not** Axis Technology Integration Partners (TIP).

## Two signing routes

Axis offers two portals, depending on your relationship with Axis:

| Route | Who | Portal |
|-------|-----|--------|
| **ACAP Service Portal** | **TIP partners** | <https://www.axis.com/partner_pages/compatible_applications/> |
| **ACAP Signing Service** | **Non-TIP** (anyone with an axis.com account) | <https://www.axis.com/support/acap-signing> |

Both return a signed `.eap`. The Service Portal additionally handles app registration, licensing, and device-compatibility declaration; the Signing Service does signing only. TIP builds carry an `appId` (assigned in the portal) in addition to `vendor`/`vendorId`; the non-TIP route uses `vendor`/`vendorId` only.

## Prerequisite that trips people up: manifest schema **must be v2.x**

The **Signing Service rejects manifest schema v1.x.** Schema `2.0.0` arrived in **ACAP Native SDK 12.10 / AXIS OS 12.10**, so before you can sign you must:

- build against **`axisecp/acap-native-sdk:12.10` or newer**,
- set **`"schemaVersion": "2.0.0"`** (or a later 2.x),
- **remove `embeddedSdkVersion`** — it was removed in schema 2.0.0 and is invalid in 2.x,
- add the fields that became **required** in the 2.x `setup` block: **`vendorId`** and **`compatibleOsVersions`**.

> ⚠️ **Device-compatibility tradeoff:** a schema-2.x package installs only on **AXIS OS 12.10+** devices. If you must support older firmware, you'll need a separate v1.x (unsigned, or TIP-signed for 9.20+) build — you can't sign a v1.x package through the non-TIP service.

### Minimal v2.x `setup` block

```json
{
    "schemaVersion": "2.0.0",
    "acapPackageConf": {
        "setup": {
            "appName": "my_app",
            "friendlyName": "My App",
            "vendor": "My Company",
            "vendorId": "1234567890",
            "runMode": "respawn",
            "version": "1.0.0",
            "compatibleOsVersions": [ { "max": "13" } ]
        }
    }
}
```

- `vendorId` — **assigned to you by the Signing Service**; the value above is a placeholder. Copy your real one from the service's welcome page before signing. It is **case-sensitive** and must match `vendor`.
- `compatibleOsVersions` — an array of `{ "min": …, "max": … }` range objects (both bounds optional). `[ { "max": "13" } ]` allows everything up to AXIS OS 13.
- `architecture` is **not** a manifest field — it stays a build arg (`ARCH`) / Dockerfile `FROM` tag.

### What `acap-build` does to your manifest (verified, SDK 12.11)

Building a source manifest that declares `"schemaVersion": "2.0.0"` and `"compatibleOsVersions": [ { "max": "13" } ]`, then inspecting the packaged `.eap` (`tar -xOzf app.eap manifest.json`), shows the build rewrites two things:

- **`schemaVersion` is normalized up to the SDK's native schema** — `2.0.0` came out as `2.2.0` on the 12.11 SDK. Declare any 2.x; the toolchain sets the exact version. (So you don't need to match the SDK's schema number exactly — just be in the 2.x range.)
- **A `min` is injected into `compatibleOsVersions` equal to the build SDK's firmware version** — declaring only `max: "13"` produced `{ "min": "12.11.72", "max": "13" }`. **The effective install floor is the AXIS OS version of the SDK image you build with.** To support a lower minimum (e.g. 12.10 devices), build with that older SDK image; you can't widen the floor below your SDK by editing the manifest.

## Signing steps (non-TIP — ACAP Signing Service)

1. Go to <https://www.axis.com/support/acap-signing> → **PROCEED TO LOGIN**, sign in with your axis.com account.
2. Accept terms: tick **"I have read and understood"** → **AGREE**.
3. If prompted to **allow local network access**, review the prompt and allow it. *(Verify what it's requesting on-screen.)*
4. On the welcome page, copy your **`vendor`** and **`vendorId`** from the shown manifest example.
5. Put those values in `app/manifest.json` (case-sensitive) and **rebuild** the `.eap` so they're baked in.
6. **CHOOSE YOUR EAP FILE TO SIGN** (or drag-and-drop) → select the rebuilt `.eap` → **SIGN THE ACAP**.
7. The **signed `.eap` downloads automatically** on success.

## Install and verify

Install the signed package the normal way — VAPIX `upload.cgi` or the device UI. Because it's signed, you do **not** need the `AllowUnsigned` toggle:

```bash
curl --request POST --anyauth -u root:<password> \
  --form 'file=@my_app_1_0_0_aarch64.eap;type=application/octet-stream' \
  "https://<device-ip>/axis-cgi/applications/upload.cgi"
```

To confirm the package really is signed before/after upload, the signed `.eap` contains a signature blob the plain build doesn't:

```bash
tar -tzf my_app_1_0_0_aarch64.eap | grep -i sig   # signed package lists a signature file
```

## How the signature works

Axis signs with **SHA-512 + a 4096-bit RSA key** held in an HSM; devices carry the matching public key and verify the signature at install time. Signatures are honored on **AXIS OS 9.20+**; older devices ignore them. This is why the signing operation is server-side only — you never hold a private key.

## References

- [ACAP application signing (service portal how-to)](https://developer.axis.com/acap/how-to-guides/service-portal/acap-application-signing/) — the TIP (Service Portal) and non-TIP (Signing Service) flows, the schema-v2.x requirement, and the crypto details.
- [ACAP Signing Service](https://www.axis.com/support/acap-signing) — the non-TIP signing portal (login required).
- [Manifest schema v2 — field descriptions](https://developer.axis.com/acap/reference/manifest-schemas/manifest-v2/schema-field-descriptions-v2.0.0/) — the required `setup` fields (`vendorId`, `compatibleOsVersions`) and the removal of `embeddedSdkVersion`.
- [ACAP 12.0 release notes](https://developer.axis.com/acap/release-notes/12.0/) — signing required by default from AXIS OS 12.0.
- Related: [enable-unsigned-apps.md](./enable-unsigned-apps.md) (the unsigned-install toggle) and [acap-manifest-gotchas.md](./acap-manifest-gotchas.md) (schema/SDK coupling).
