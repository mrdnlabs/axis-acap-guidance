# Local VAPIX Authentication from an ACAP

How to make authenticated VAPIX calls from within an ACAP to the same device, without storing credentials.

---

## The Pattern

Use D-Bus to request runtime VAPIX service account credentials, then call VAPIX on the loopback interface. Credentials are held in memory only and re-fetched on each ACAP startup.

### 1. Declare the D-Bus method in manifest.json

```json
"resources": {
    "dbus": {
        "requiredMethods": [
            "com.axis.HTTPConf1.VAPIXServiceAccounts1.GetCredentials"
        ]
    }
}
```

### 2. Retrieve credentials at runtime

```c
const char* bus_name = "com.axis.HTTPConf1";
const char* object_path = "/com/axis/HTTPConf1/VAPIXServiceAccounts1";
const char* interface_name = "com.axis.HTTPConf1.VAPIXServiceAccounts1";
const char* method_name = "GetCredentials";

GDBusConnection *connection = g_bus_get_sync(G_BUS_TYPE_SYSTEM, NULL, &error);
GVariant *result = g_dbus_connection_call_sync(
    connection, bus_name, object_path, interface_name, method_name,
    g_variant_new("(s)", username),
    NULL, G_DBUS_CALL_FLAGS_NONE, -1, NULL, &error);
```

The service returns credentials as a colon-separated `username:password` string.

### 3. Call VAPIX on loopback

Always use a loopback address — never the device's external IP.

```c
// Use 127.0.0.1 or 127.0.0.12 (Axis virtual host for service accounts)
curl_easy_setopt(curl, CURLOPT_URL, "http://127.0.0.1/axis-cgi/...");
curl_easy_setopt(curl, CURLOPT_HTTPAUTH, CURLAUTH_DIGEST);
curl_easy_setopt(curl, CURLOPT_USERPWD, credentials);
```

### 4. Clean up

Free credentials from memory when done. Do not cache to disk.

---

## Loopback Address

Axis documents **`127.0.0.12`** as *the* virtual host for the service-account credentials returned by `GetCredentials`, and the official `vapix` example hardcodes it (`http://127.0.0.12/axis-cgi/...`). These credentials are bound to that virtual host — per the docs they are "only valid on the local virtual host (127.0.0.12)." **Use `127.0.0.12`** for service-account calls.

> ⚠️ **On `127.0.0.1` / `127.0.1.1` fallbacks:** standard loopback addresses are **not** documented as valid for these credentials, and there's no Axis guarantee they route to the virtual host that accepts them. Earlier versions of this note recommended probing all three; that was an empirical workaround observed against a *non-standard* endpoint (the AOA app's `/local/objectanalytics/control.cgi`, which is served by the AOA ACAP, not core VAPIX) — see [aoa-api-patterns.md](./aoa-api-patterns.md). For ordinary `axis-cgi` calls, prefer `127.0.0.12` and treat a `401` there as a real auth problem (e.g. stale credentials, wrong username), not a reason to fall back to another loopback address.
>
> Note also: if a global device proxy is configured, `127.0.0.12` must be added to the device's "No proxy" list.

---

## Common Pitfalls

- **Do not use the device's external LAN IP** for local VAPIX calls. The ACAP will break on any device with a different IP.
- **Re-fetch credentials on each startup.** The D-Bus credentials are transient and should not be persisted.
- **The D-Bus API version has changed across AXIS OS releases.** Per the official `vapix` example README, *"the format of the D-Bus API to get VAPIX credentials changed in 11.8."* The current signature passes a username string — `g_variant_new("(s)", username)`. Check the official VAPIX SDK example for the signature on your target firmware.
- **These credentials are valid only for loopback.** They cannot be used to authenticate against a remote device.

---

## References

- [VAPIX access for ACAP applications](https://developer.axis.com/acap/develop/VAPIX-access-for-ACAP-applications/) — the canonical source. Documents the `com.axis.HTTPConf1.VAPIXServiceAccounts1.GetCredentials` D-Bus method, the `resources.dbus.requiredMethods` manifest declaration, the `(s)` username signature, and states verbatim that the credentials *"are only valid on the local virtual host (127.0.0.12)"* and must be *"re-fetched each time the ACAP application starts … only be kept in memory … not stored in any file."* Also notes `127.0.0.12` must be on the device's "No proxy" list.
- [Official SDK example: `acap-native-sdk-examples/vapix`](https://github.com/AxisCommunications/acap-native-sdk-examples/tree/main/vapix) — working reference implementation; its README states the D-Bus credential format *"changed in 11.8."*
