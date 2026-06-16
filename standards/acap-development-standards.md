# ACAP Development Standards

> **What:** Technical standards and conventions for building Axis ACAP applications — project structure, SDK usage, manifest conventions, build/deploy workflow, and security requirements.
>
> **Who:** Developers (human or agentic) actively building or modifying an ACAP project.
>
> **When:** Reference throughout development. Consult before writing code (for structure and SDK choices) and before finishing (for the "What NOT to Do" checklist).
>
> **Related documents:**
> - [acap-project-suitability.md](./acap-project-suitability.md) — Gate 0: decide whether this project should be an ACAP at all (do this first)
> - [acap-project-go-to-prompt.md](./acap-project-go-to-prompt.md) — Reusable prompt to kick off agentic development sessions
> - [acap-security-audit.md](./acap-security-audit.md) — Security and quality audit prompts/checklists (use before publishing)
> - [../guides/](../guides/) — Hard-won platform knowledge: VAPIX patterns, manifest/build gotchas, and device-specific notes

---

## 1. Project Scope

ACAPs developed under these standards are **single-purpose utility applications** that solve one well-defined operational problem on Axis devices.

**In scope:**
- A targeted tool that can be described in one sentence (e.g., "Displays AOA counter data as a large overlay")
- Automation of a specific device workflow or integration
- A focused utility that extends device capability in a narrow, well-understood way

**Litmus test:** If you can't explain what the app does in one sentence, it's too broad. If it needs a product manager, it's not a utility ACAP.

---

## 2. Safety and Risk

ACAPs developed under these standards **must not** be used in contexts where a malfunction could cause:

- Risk to human safety or physical security
- Significant financial loss
- Loss of critical evidence or data
- Disruption to emergency or life-safety systems

These are utility and convenience applications. If the ACAP stops working, the worst outcome should be loss of a convenience feature — not a safety incident. Safety-critical functionality must remain in the device firmware or a certified system, not in a custom ACAP.

---

## 3. Licensing

All projects use the **MIT License** by default.

- Include a `LICENSE` file in the project root with the MIT License text
- Include a copyright line: `Copyright (c) <year> <author/org>`
- If third-party dependencies use other licenses (e.g., CivetWeb uses MIT, some Axis examples use Apache 2.0), document them in the LICENSE file or a THIRD-PARTY-LICENSES section
- Do not use copyleft licenses (GPL, AGPL) unless there is a specific reason

---

## 4. Security

### 4.1 General Principles

- **No new attack surface.** The ACAP should not weaken the device's security posture.
- **Leverage Axis OS security.** Use built-in authentication, TLS, and access control rather than implementing your own.
- **Minimize network exposure.** Do not open additional ports unless absolutely necessary.
- **Validate all external input.** Treat query parameters, API request bodies, and parameter values as untrusted.
- **No hardcoded credentials.** Never embed usernames, passwords, API keys, or tokens in source code.

### 4.2 Web Server Components

**Use reverse proxy or FastCGI unless there is a compelling reason not to.**

| Approach | When to Use |
|----------|-------------|
| **FastCGI** (`httpConfig`) | Preferred for simple request/response CGI endpoints. Runs inside Apache — gets TLS, auth, and access control automatically. |
| **Reverse Proxy** (`reverseProxy`) | Use when the app needs a persistent HTTP server (WebSocket, long-polling, streaming, or complex routing). Apache proxies to localhost; TLS and auth still handled by the device. |
| **Raw HTTP server** | Avoid. Only acceptable for development/debugging or when the above mechanisms are technically impossible. Document the justification. |

Both FastCGI and reverse proxy inherit device authentication (digest auth), TLS termination, and CSP headers. A raw HTTP server on an open port bypasses all of these.

**Reverse proxy gotchas (from hard-won experience — see the [reverse proxy guide](../guides/acap-reverse-proxy-guide.md) and [WSL build pitfalls](../guides/acap-wsl-build-pitfalls.md) for the full detail):**
- Apache forwards the full URI unchanged — register handlers at `/local/<appName>/<apiPath>/...`
- WSL 777 file permissions silently prevent proxy rule creation — always `chmod 644`/`755` in the Dockerfile
- Old `.eap` files in the build context cause manifest field merging — use `.dockerignore`

### 4.3 Credential Storage

**For local VAPIX calls (same device):**
- Use D-Bus `com.axis.HTTPConf1.VAPIXServiceAccounts1.GetCredentials` for runtime credentials
- Call VAPIX on loopback (`127.0.0.1` or `127.0.0.12`)
- Re-fetch credentials on each ACAP startup; do not cache to disk
- Declare the required D-Bus method in `manifest.json` under `resources.dbus.requiredMethods`

**For remote device credentials (controlling another camera or external endpoint):**

The Axis ACAP SDK provides **no encrypted/keystore API**. All persistent options leave the value as cleartext on the device's flash; a root compromise of the device exposes the value regardless of which option you pick. The choice is about closing the `param.cgi` leak path and the settings-UI / audit-log leaks — not about at-rest encryption.

Ranked by Axis documentation strength:

**(a) AXParameter `password` type — documented, masks `param.cgi`.** Declare the field in `manifest.json` with `"type": "password:maxlen=N"` (or just `"password"` with no length cap). The runtime then:
- Returns `******` from `param.cgi?action=list` instead of the value
- Renders a masked input in the ACAP settings UI
- Masks the value in the AXIS OS 12.7+ audit log
- Still returns the real value to `ax_parameter_get()` inside the ACAP, so outbound auth works unchanged

This is the only option Axis documents (`ax_parameter.h`, present since the ACAP 3 era). It does not require new storage code. **Default to this.**

**(b) File in `localdata/` with `fchmod(0600)`.** Invisible to `param.cgi` entirely. Owned by APPUSR. Useful only if you also need to bypass AXParameter (e.g. for rotation logic, multi-field bundles, or because the value exceeds `maxlen` constraints). Requires building your own settings UI plumbing. No Axis doc precedent — only used by some sibling projects (e.g. SfH).

**(c) In-memory only, re-entered at every restart.** Endorsed for *local* VAPIX (`vapix/` SDK example), but operator UX is poor for fields the user expects to "set and forget." Use only when there's a human in the loop at every boot.

**Common rules regardless of option:**
- Always use HTTPS/TLS for outbound calls
- Never log credentials, even at debug level
- Never put a default password in `manifest.json` `paramConfig` defaults
- Do not echo the password back via your own API; expose a `…Configured` boolean instead, plus an explicit clear path
- Validate inputs (length, character class) before storing
- **`hidden:string` is NOT a security control** — it only hides the field in the settings UI, `param.cgi` still returns the value. See [`acap-manifest-gotchas.md`](../guides/acap-manifest-gotchas.md).

**Migration note:** when upgrading an existing deployment from `String` to `password:maxlen=N`, the old cleartext value persists on flash until the next write to the param. Rotate any password that was previously exposed via `param.cgi` as a separate step after upgrading.

**For development and deployment credentials (`.env.devices`):**

`.env.devices` is a **development-time file** used by the developer or agentic tool to access test devices for building, deploying, and verifying ACAPs. It is not part of the ACAP itself and is never deployed to the device.

- Store device IPs, usernames, and passwords in `.env.devices` or similar local-only file
- Add to `.gitignore` — never commit
- Prefer temporary credentials created for the development session (see the [go-to prompt](./acap-project-go-to-prompt.md) for agentic credential handling)
- Use digest auth (`--anyauth`) for VAPIX calls in deployment scripts

### 4.4 Input Validation

- Sanitize and validate all query parameters, POST bodies, and parameter callback values
- HTML-encode any user-supplied data before rendering in web UI responses (the FastCGI SDK example does NOT do this — do better)
- Use parameterized/structured APIs rather than string concatenation for VAPIX calls
- Never pass user input to `system()`, `popen()`, or shell commands
- Validate file paths to prevent traversal attacks (reject `..`, absolute paths from user input)

### 4.5 Access Control

- Set the `"access"` field in manifest `reverseProxy` or `httpConfig` entries to the most restrictive level needed:
  - `"admin"` for configuration endpoints
  - `"operator"` for operational controls
  - `"viewer"` for read-only status
  - `"anonymous"` only when explicitly required and after careful consideration — note this level is valid for `reverseProxy` entries only; `httpConfig` (FastCGI) supports `admin`/`operator`/`viewer` but not `anonymous`
- Separate admin and viewer endpoints into different proxy/CGI entries with different access levels

---

## 5. Axis SDK and Technology Preferences

**Strong preference for Axis-provided technologies over rolling your own.**

| Need | Use This | Not This |
|------|----------|----------|
| Web UI auth/TLS | Reverse proxy or FastCGI | Custom auth middleware |
| App parameters | AXParameter library | Custom config files |
| Event system | AXEvent library | Custom IPC/polling |
| Video access | VDO API | Direct device file access |
| ML inference | Larod API | Custom TensorFlow runtime |
| Overlays | Axoverlay API | Direct framebuffer |
| Storage | AXStorage API | Hardcoded paths |
| Serial ports | AXSerialPort API | Direct `/dev/tty*` |
| Local VAPIX auth | D-Bus GetCredentials | Stored username/password |
| HTTP client (TLS) | curl + openssl (as in SDK examples) | Custom socket code |

When the SDK provides a capability, use it. It handles firmware version differences, device quirks, and security boundaries that custom code will not.

---

## 6. Project Structure

```
my-acap/
├── app/
│   ├── main.c (or primary source files)
│   ├── manifest.json
│   ├── Makefile
│   ├── LICENSE
│   └── html/           (if web UI)
│       └── index.html
├── Dockerfile
├── .dockerignore
├── .gitignore
├── .env.devices         (development-time device credentials, git-ignored — not deployed to device)
├── README.md
└── LICENSE
```

### Dockerfile Template (WSL-safe)

```dockerfile
ARG ARCH=aarch64
ARG VERSION=12.9.0
ARG UBUNTU_VERSION=24.04

FROM axisecp/acap-native-sdk:${VERSION}-${ARCH}-ubuntu${UBUNTU_VERSION}

COPY ./app /opt/app/
WORKDIR /opt/app

# Fix WSL file permissions (no-op on native Linux)
RUN find . -type f -exec chmod 644 {} + && \
    find . -type d -exec chmod 755 {} +

RUN . /opt/axis/acapsdk/environment-setup* && acap-build ./
```

### .dockerignore

```
*.eap
*.o
.env.devices
.git
docs
```

### .gitignore

```
*.eap
*.o
.env.devices
build/
```

---

## 7. Manifest Conventions

- `schemaVersion`: Must match SDK version (check `/opt/axis/acapsdk/axis-acap-manifest-tools/schema/schemas/v1/` in container)
- `embeddedSdkVersion`: `"3.0"` — this is the framework version, not the SDK version. (Note: this field was **removed in manifest schema 2.0.0 / SDK 12.10**; omit it entirely when targeting `schemaVersion` ≥ `2.0.0`. For v1.x schemas, `"3.0"` is correct.)
- `runMode`: `"respawn"` for production apps (auto-restart on crash), `"once"` (start at boot, no respawn), or `"never"` for examples/testing
- `settingPage`: `"index.html"` — note singular "setting", not "settings"
- `appName`: lowercase with underscores, no hyphens
- `vendor`: Your organization name (not placeholder text)
- `vendorUrl`: A real URL or omit entirely — not `https://example.invalid`

---

## 8. Version Control

All projects must be version-controlled with Git.

- Initialize a Git repository at project start
- Write meaningful commit messages describing *why*, not just *what*
- Tag releases with semantic versioning: `v1.0.0`, `v1.1.0`, etc.
- Never commit: `.eap` files, `.env.devices`, build artifacts, credentials
- Include `.gitignore` from the start
- Keep the repository focused — one ACAP per repository

---

## 9. README Requirements

Every project must include a `README.md` with at minimum:

1. **Title and one-sentence description** — what the app does
2. **Prerequisites** — target architecture, AXIS OS version, required device features
3. **Build instructions** — exact Docker commands to build the `.eap`
4. **Installation instructions** — how to deploy to a device
5. **Configuration** — what parameters exist and what they control
6. **Known limitations** — firmware version constraints, device model requirements, known issues
7. **License** — reference to the LICENSE file

Optional but encouraged:
- Screenshots of the web UI if one exists
- Troubleshooting section for common deployment issues

For non-trivial apps, include a brief **"How it works"** section (3-5 sentences) describing the data flow and key design decisions. This helps an agentic tool or new developer orient quickly without reading every source file. Example:

> *"The ACAP polls AOA `getAccumulatedCounts` every N ms, extracts the configured field, and writes it to both a custom overlay (via axoverlay) and a dynamic text slot (via dynamicoverlay.cgi). Configuration is stored in ax_parameter. The web UI is served via CivetWeb behind the Apache reverse proxy."*

---

## 10. Design for Testability

Before writing code, think about how you will verify that the application works correctly on-device. Design the application so that its behavior can be observed and validated — by a human, an agentic tool, or both.

**Design principles:**

- **Expose status via API.** If the app does something (polls a count, plays audio, sends an event), expose the current state through an API endpoint (e.g., `/api/status`). This lets an agentic tool or script verify behavior without relying on visual inspection.
- **Make key behaviors observable.** If the app writes an overlay, also make the overlay text available via API. If it triggers an event, make the last event timestamp queryable. If it controls another device, expose the last command sent and the response received.
- **Use structured logging.** Log key state transitions to syslog with consistent, parseable prefixes (e.g., `[myapp] poll: count=8 scenario=2`). This makes log-based verification straightforward.
- **Separate concerns.** Keep business logic (what to do) separate from I/O (how to talk to the device). This makes it easier to reason about correctness and to test individual pieces.
- **Make configuration changes take effect without reinstalling.** Use `ax_parameter_get()` at the point of use (not just at startup) for settings that an operator or test script might change during a session.

**Before starting implementation, answer these questions:**

1. After deploying, how will I confirm the app is running and healthy?
2. How will I verify the core feature is working (not just that the process started)?
3. If the app interacts with external APIs or devices, how will I confirm those interactions are happening correctly?
4. Can an agentic tool with browser control and API access verify all of the above without human visual inspection?
5. What could go wrong silently (no error, but wrong behavior), and how would I detect it?

If any answer is "I'd have to look at the screen" or "I'd have to manually check," consider adding an observable endpoint or log entry that makes it programmatically verifiable.

---

## 11. Build and Deployment

- Build exclusively via Docker using the official `axisecp/acap-native-sdk` images
- Support at minimum `aarch64`; add `armv7hf` if targeting older devices
- Deploy via VAPIX `upload.cgi` or the device web UI
- Test on real hardware — simulators do not cover all device behaviors
- Verify the reverse proxy is active after deployment (check response headers)
- Document the tested device models and AXIS OS versions

---

## 12. Agentic Development Workflow

When using AI-assisted tools to develop ACAPs, the following prompt should be used or adapted at the start of each project. See `acap-project-go-to-prompt.md` for the full reusable prompt.

Key workflow requirements for agentic development:
- **Research first:** Check latest SDK docs, VAPIX APIs, and GitHub examples before writing code
- **Inspect before assuming:** Read the existing workspace/repo before making changes
- **Test on real devices:** Treat device deployment and verification as part of the task
- **Prefer on-camera solutions:** Avoid host-side bridges unless explicitly requested
- **Document learnings:** Save findings, caveats, and platform-specific notes
- **Audit before finishing:** Run the security audit checklist (see below) and a publish-readiness check
- **Handle device credentials carefully:** Store in `.env.devices` (git-ignored), prefer temporary/dev-only credentials created for the session, and rotate or remove them when done. See the go-to prompt for full guidance.

---

## Quick Reference: What NOT to Do

| Don't | Do Instead |
|-------|------------|
| Open a raw HTTP port with no auth | Use reverse proxy or FastCGI |
| Store credentials in a `String` param (cleartext via `param.cgi`) | Use `password:maxlen=N` type, or hold in memory (see §4.3) |
| Hardcode device IPs or credentials in source | Use ACAP parameters for runtime config; `.env.devices` for development/deployment access |
| Use `system()` or `popen()` with user input | Use structured APIs |
| Build a "platform" or "framework" | Solve one specific problem |
| Skip device testing | Deploy, test, verify on real hardware |
| Use `sprintf()` without bounds | Use `snprintf()` or `g_strdup_printf()` |
| Commit `.eap` files or credentials | Add to `.gitignore` from day one |
| Use placeholder vendor metadata | Use real values or omit |
| Implement custom auth | Leverage Axis OS auth via proxy/FastCGI |

---

## References / Further reading

Official Axis sources for the technical claims in this document. The platform moves quickly — always confirm against the docs for your target SDK / AXIS OS version.

- [Supported APIs — ACAP Native SDK](https://developer.axis.com/acap/reference/supported-apis/) — the SDK preference table in §5 (VDO, AXEvent, Axoverlay, Larod, AXStorage, AXSerialPort, AXParameter, plus bundled libs: FastCGI, OpenSSL, Curl, Jansson, Cairo).
- [Manifest schemas — overview & version history](https://developer.axis.com/acap/reference/manifest-schemas/general-info/) and the [v2.0.0 field descriptions](https://developer.axis.com/acap/reference/manifest-schemas/manifest-v2/schema-field-descriptions-v2.0.0/) — `schemaVersion`/`runMode`/`settingPage`/`paramConfig`, the `reverseProxy` vs `httpConfig` `access` values (only `reverseProxy` allows `anonymous`), and the removal of `embeddedSdkVersion` in 2.0.0 (§4.2, §4.5, §7).
- [Web server via reverse proxy](https://developer.axis.com/acap/develop/web-server-via-reverse-proxy/) — Apache reverse proxy inherits authentication and TLS (§4.2).
- [VAPIX access for ACAP applications](https://developer.axis.com/acap/develop/VAPIX-access-for-ACAP-applications/) — D-Bus `GetCredentials`, the `127.0.0.12` virtual host, keep-in-memory guidance (§4.3).
- [ACAP 12.7 release notes](https://developer.axis.com/acap/release-notes/12.7/) — `password` / `writeonly` parameter values are masked in the audit log (§4.3).
- [AXIS OS Hardening Guide](https://help.axis.com/en-us/axis-os-hardening-guide) — device security baseline an ACAP should not weaken (§4).
- [acap-native-sdk-examples](https://github.com/AxisCommunications/acap-native-sdk-examples) (Apache-2.0) — official example code referenced throughout these standards.
