# ACAP Reverse Proxy: Complete Guide

How the Apache reverse proxy works for ACAP web UIs on AXIS OS 12.9+, and every pitfall we found.

---

## How It Works

The `reverseProxy` block in `manifest.json` tells the device to create Apache ProxyPass rules that forward requests from the camera's web server (port 80/443) to your CivetWeb server on localhost.

```json
"configuration": {
    "settingPage": "index.html",
    "reverseProxy": [
        {
            "apiPath": "api",
            "target": "http://localhost:8080",
            "access": "admin"
        }
    ]
}
```

This creates a proxy at: `http://<DEVICE_IP>/local/<appName>/<apiPath>/...`

Your web UI gets TLS, digest auth, access control, CSP headers — everything the device admin configures — for free.

---

## Critical: Apache Does NOT Strip the Path Prefix

**This is the most important thing to know.** Apache forwards the **full original URI** to CivetWeb. It does NOT strip `/local/<appName>/<apiPath>/`.

A request to:
```
GET /local/myapp/api/status
```
CivetWeb receives:
```
GET /local/myapp/api/status    ← full path, unchanged
```

**NOT** `/status` or `/api/status` as you might expect.

The official Axis `web-server` example masks this completely because it registers a `"/"` catch-all handler that matches any URI.

### Fix: Register handlers at the full proxy path

```c
#define PROXY_PFX "/local/myapp/api"

/* Direct access (port 8080) */
mg_set_request_handler(ctx, "/status", handler_status, NULL);
mg_set_request_handler(ctx, "/test$",  handler_test,   NULL);

/* Proxy access (port 80/443 via Apache) */
mg_set_request_handler(ctx, PROXY_PFX "/status", handler_status, NULL);
mg_set_request_handler(ctx, PROXY_PFX "/test$",  handler_test,   NULL);
```

### Web UI JavaScript

Use the proxy path for API calls — this ensures auth/TLS coverage:

```js
const API_BASE = `/local/myapp/api`;
const data = await fetch(`${API_BASE}/status`);
```

For resilience, probe the proxy and fall back to direct:

```js
let API_BASE = `/local/myapp/api`;
try {
    const r = await fetch(`${API_BASE}/test`, { credentials: 'same-origin' });
    if (!r.ok) throw new Error(r.status);
} catch {
    API_BASE = `http://${window.location.hostname}:8080`;
}
```

---

## WSL File Permissions Break the Proxy

**If you build on WSL2 with source files on `/mnt/c/`**, all files get 777 permissions. The ACAP installer on the device **silently refuses to create the Apache proxy rules** when EAP files have 777 permissions. The app installs and runs fine — only the proxy is missing. No error is logged.

### Fix: chmod in Dockerfile

```dockerfile
COPY ./app /opt/app/
WORKDIR /opt/app

# Fix WSL 777 permissions → standard Unix perms
RUN find . -type f -exec chmod 644 {} + && \
    find . -type d -exec chmod 755 {} +

RUN . /opt/axis/acapsdk/environment-setup* && acap-build ./
```

This is a no-op on Linux/macOS and harmless to always include.

---

## .dockerignore Must Exclude *.eap

If old `.eap` files exist in `app/`, `COPY ./app /opt/app/` sends them into the Docker build context. `acap-build` silently reads the manifest.json inside old EAP archives and **merges fields** (like `paramConfig`, `user`, `friendlyName`) into the new build's manifest — even if you removed those fields from your source manifest.json.

```dockerignore
# .dockerignore
*.eap
*.o
.env.devices
.git
docs
```

(Use `*.eap` rather than `app/*.eap` so root-level builds are also excluded.)

---

## Manifest Configuration

```json
{
    "schemaVersion": "1.9.0",
    "acapPackageConf": {
        "setup": {
            "appName": "myapp",
            "vendor": "My Company",
            "embeddedSdkVersion": "3.0",
            "runMode": "respawn",
            "version": "1.0.0"
        },
        "configuration": {
            "settingPage": "index.html",
            "reverseProxy": [
                {
                    "apiPath": "api",
                    "target": "http://localhost:8080",
                    "access": "admin"
                }
            ]
        }
    }
}
```

### Field notes

| Field | Value | Notes |
|-------|-------|-------|
| `schemaVersion` | e.g. `"1.9.0"` | Must be a schema your SDK ships. The mapping is SDK-specific (`1.9.0` ← SDK 12.8; `2.0.0` ← SDK 12.10). Check the container, don't hardcode from memory |
| `embeddedSdkVersion` | `"3.0"` | ACAP **framework** version, NOT SDK version. Use `"3.0"` for v1.x schemas; **removed in schema 2.0.0** — omit it on SDK 12.10+ |
| `runMode` | `"respawn"`, `"once"`, or `"never"` | All work with proxy. `"respawn"` auto-restarts on crash |
| `apiPath` | any string | Becomes part of the URL. Can contain `/` (e.g., `"api/v1"`) |
| `target` | `"http://localhost:<port>"` | Your CivetWeb listen port |
| `access` | `"admin"`, `"operator"`, `"viewer"`, `"anonymous"` | Auth level required |

### Multiple access levels

Use multiple entries pointing to the same target with different paths and access levels:

```json
"reverseProxy": [
    { "apiPath": "api/admin",     "target": "http://localhost:8080", "access": "admin" },
    { "apiPath": "api/viewer",    "target": "http://localhost:8080", "access": "viewer" },
    { "apiPath": "api/anonymous", "target": "http://localhost:8080", "access": "anonymous" }
]
```

---

## HTTPCGIPATHS in package.conf

`HTTPCGIPATHS` is a `package.conf` variable from the pre-manifest era; it maps to the `httpConfig` (FastCGI/CGI) mechanism, **NOT** `reverseProxy`. It is expected to be empty when using `reverseProxy`. The device reads `manifest.json` directly for proxy configuration.

---

## Debugging Checklist

If your reverse proxy isn't working:

1. **Check file permissions in EAP**: `tar -tvzf myapp.eap` — files should be 644, dirs 755
2. **Verify proxy exists**: Compare response format for `/local/myapp/api/test` vs `/local/myapp/notapi/test`. CivetWeb returns `text/plain` 404; Apache returns `text/html` 404
3. **Check what path CivetWeb receives**: Add a catch-all handler that returns `ri->local_uri` in the response
4. **Check for stale EAPs in build context**: Ensure `.dockerignore` excludes `*.eap`
5. **Try direct access**: `curl http://<IP>:8080/test` — if this works, the issue is proxy setup, not the app

---

## Reference: Official Examples Using reverseProxy

Only 2 official examples exist:

1. **[`acap-native-sdk-examples/web-server/reverse-proxy-using-fixed-port`](https://github.com/AxisCommunications/acap-native-sdk-examples/tree/main/web-server/reverse-proxy-using-fixed-port)** — C + CivetWeb, single proxy entry, catch-all `"/"` handler (this example was previously named just `web-server`)
2. **[`acap-rs/apps/reverse_proxy`](https://github.com/AxisCommunications/acap-rs/tree/main/apps/reverse_proxy)** — Rust, 4 proxy entries with different access levels, also uses `httpConfig`

Both use `runMode: "never"` and port 2001, but these are not requirements. (Note: the upstream `reverse-proxy-using-fixed-port` manifest has since moved to `schemaVersion 2.0.0` with no `embeddedSdkVersion` — the `1.9.0` sample above is still valid for SDK 12.8–12.9.)

---

## References

- [Web server via reverse proxy](https://developer.axis.com/acap/develop/web-server-via-reverse-proxy/) — official overview: the app's API is exposed through the AXIS OS Apache server, which routes internally to a small web server in the app, inheriting Apache authentication and TLS.
- [Manifest schema — field descriptions](https://developer.axis.com/acap/reference/manifest-schemas/manifest-v1/schema-field-descriptions-v1.10.0/) — `reverseProxy` objects and the `apiPath` / `target` / `access` fields; `httpConfig` ("list of web server configuration objects"), which is distinct from `reverseProxy`.
- [Example: `web-server/reverse-proxy-using-fixed-port`](https://github.com/AxisCommunications/acap-native-sdk-examples/tree/main/web-server/reverse-proxy-using-fixed-port) — C + CivetWeb; registers handlers on the full `/local/<appName>/<apiPath>` path (the source is the practical proof that Apache forwards the full URI).
- [Example: `acap-rs/apps/reverse_proxy`](https://github.com/AxisCommunications/acap-rs/tree/main/apps/reverse_proxy) — Rust; four `reverseProxy` entries demonstrating `admin`/`operator`/`viewer`/`anonymous`, plus an `httpConfig` entry.

> **Empirical / observed (no official source):** the headline "Apache forwards the full URI unchanged" is verifiable from the example's source (it registers a `"/"` catch-all), not from Axis prose; likewise the WSL-777-breaks-the-proxy behavior, `acap-build` merging manifest fields from a stale `.eap`, the CivetWeb-vs-Apache 404 content-type heuristic, and the `HTTPCGIPATHS` `package.conf` variable name are field notes, not documented contracts.
