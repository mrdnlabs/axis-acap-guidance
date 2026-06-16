# ACAP Security Audit Prompts and Checklists

> **What:** A structured set of audit prompts and manual checklists for reviewing ACAP application code for security vulnerabilities, packaging issues, and publish readiness. Each audit can be used as a standalone checklist or fed directly to an agentic code review tool.
>
> **Who:** Developers, reviewers, or agentic tools performing a security and quality review of an ACAP project.
>
> **When:** Before publishing or deploying an ACAP to production devices. Can also be run periodically during development to catch issues early. The "Full Audit" prompt at the bottom runs all six checks in sequence.
>
> **Related documents:**
> - [acap-development-standards.md](./acap-development-standards.md) — The standards these audits verify compliance with
> - [acap-project-suitability.md](./acap-project-suitability.md) — Gate 0: suitability assessment (should have been completed before development began)
> - [acap-project-go-to-prompt.md](./acap-project-go-to-prompt.md) — Development session prompt (references this audit as a final step)
> - [../guides/](../guides/) — Hard-won platform knowledge: VAPIX patterns, manifest/build gotchas, and device-specific notes

---

## Audit 1: Network Exposure and Authentication

**Prompt for agentic review:**

> Review this ACAP project for network exposure and authentication issues. Check:
>
> 1. Does the app open any TCP/UDP ports? List every port and what listens on it.
> 2. For each port: is it accessible only on localhost, or on all interfaces?
> 3. Is a reverse proxy or FastCGI configured in `manifest.json`? If not, why?
> 4. If a reverse proxy is configured, does every `reverseProxy` entry have an appropriate `"access"` level?
> 5. Are there any endpoints accessible at `"anonymous"` access level? If so, justify each one.
> 6. If the app runs its own HTTP server (CivetWeb, etc.), can it be reached directly on its port from the network, bypassing Apache auth?
> 7. Does the web UI use the proxy path (`/local/<appName>/...`) for API calls, or does it hardcode a direct port?
>
> Flag any path where an unauthenticated network client could reach application logic.

**Manual checklist:**

- [ ] `manifest.json` uses `reverseProxy` or `httpConfig` (FastCGI) for all web endpoints
- [ ] Every proxy/CGI entry has `"access"` set to the most restrictive level needed
- [ ] No `"anonymous"` access unless explicitly justified and documented
- [ ] Internal HTTP server binds to `localhost` only (or access is acceptable if proxy fails)
- [ ] No unnecessary ports are opened
- [ ] Web UI JavaScript uses `/local/<appName>/<apiPath>/` paths, not direct `http://host:port/`

---

## Audit 2: Credential Handling

**Prompt for agentic review:**

> Review this ACAP project for credential handling issues. Check:
>
> 1. Search all source files for hardcoded passwords, API keys, tokens, or usernames. Check string literals, `#define` values, and default parameter values in `manifest.json`.
> 2. Are any credentials stored in ACAP parameters (`ax_parameter`)? These are readable via `param.cgi` in plaintext.
> 3. If the app needs local VAPIX access: does it use D-Bus `GetCredentials`, or does it store/hardcode credentials?
> 4. If the app communicates with remote devices: how are remote credentials obtained, stored, and transmitted?
> 5. Are credentials ever logged (syslog, printf, debug output)?
> 6. Are credentials cleared from memory when no longer needed?
> 7. Is `.env.devices` or equivalent in `.gitignore`?
> 8. Check git history for any previously committed credentials.
>
> Flag any credential that is hardcoded, stored in plaintext parameters, logged, or committed to version control.

**Manual checklist:**

- [ ] No hardcoded credentials in source code
- [ ] No passwords in `manifest.json` default parameter values
- [ ] Local VAPIX uses D-Bus `GetCredentials` + loopback, not stored credentials
- [ ] Remote-device credentials use `manifest.json` `"type": "password:maxlen=N"` (NOT `"String"` and NOT `"hidden:string"` — `hidden` does not mask `param.cgi`)
- [ ] `curl https://<device>/axis-cgi/param.cgi?action=list&group=root.<App>` returns `******` for every credential field, not the value
- [ ] API does not echo the password back (use a `…Configured` boolean instead) and provides an explicit clear path
- [ ] Credentials never written to syslog or debug output
- [ ] `.env.devices` and similar files are in `.gitignore`
- [ ] `git log -p --all -S 'password'` and `git log -p --all -S 'secret'` return no real credentials
- [ ] HTTPS used for all remote device communication

---

## Audit 3: Input Validation and Injection

**Prompt for agentic review:**

> Review this ACAP project for input validation and injection vulnerabilities. Check:
>
> 1. Trace every path where external input enters the application: HTTP query parameters, POST bodies, ACAP parameter callbacks, and any data read from external APIs.
> 2. For each input path: is the input validated, sanitized, or bounded before use?
> 3. Is any user input passed to `system()`, `popen()`, `exec*()`, or any shell-executing function? (This would be command injection.)
> 4. Is any user input used to construct file paths? Check for `..` traversal and absolute path injection.
> 5. Is any user input rendered in HTML responses? Check for XSS — is it HTML-encoded before output?
> 6. Is any user input used in VAPIX/CGI URL construction? Check for injection in query parameters.
> 7. Are buffer sizes enforced? Check for `sprintf()` without bounds, `strcpy()` without length checks, or unbounded `sscanf()`.
> 8. For numeric parameters: are range checks applied before use?
>
> Flag any path where unsanitized external input reaches a sensitive operation.

**Manual checklist:**

- [ ] No calls to `system()`, `popen()`, or `exec*()` with user-controlled data
- [ ] No `sprintf()` — use `snprintf()` or `g_strdup_printf()` with bounds
- [ ] No `strcpy()` — use `strncpy()` or GLib string functions
- [ ] All HTTP query parameters validated before use
- [ ] All HTML output of user data is HTML-encoded
- [ ] File paths constructed from user input reject `..` and absolute paths
- [ ] Numeric parameters have range validation
- [ ] VAPIX URL parameters are properly escaped

---

## Audit 4: Build and Packaging Hygiene

**Prompt for agentic review:**

> Review this ACAP project's build and packaging for security and correctness issues. Check:
>
> 1. Does the Dockerfile include the `chmod 644/755` fix for WSL file permissions?
> 2. Does `.dockerignore` exclude `*.eap`, `*.o`, `.env.devices`, and `.git`?
> 3. Are there any old `.eap` files in the `app/` directory that could cause manifest merging during build?
> 4. Does `manifest.json` use correct `schemaVersion` for the target SDK?
> 5. Is `embeddedSdkVersion` set to `"3.0"` (not the SDK version number)?
> 6. Are there any test-only code paths, debug endpoints, or placeholder values that should be removed before release?
> 7. Does the `.eap` package contain any files it shouldn't (credentials, debug configs, build artifacts)?
> 8. Is `runMode` set appropriately (`"respawn"` for production, `"never"` only for examples)?
>
> Flag any packaging issue that could cause silent failures or include unintended content.

**Manual checklist:**

- [ ] Dockerfile includes WSL chmod fix
- [ ] `.dockerignore` excludes `.eap`, `.o`, `.env.devices`, `.git`
- [ ] No stale `.eap` files in `app/` directory
- [ ] `schemaVersion` matches SDK version
- [ ] `embeddedSdkVersion` is `"3.0"`
- [ ] No debug endpoints or test-only code in release build
- [ ] `vendor` and `vendorUrl` are real values, not placeholders
- [ ] `runMode` is `"respawn"` for production deployments
- [ ] Line endings are LF (not CRLF) for all shell scripts and source files

---

## Audit 5: Publish Readiness

**Prompt for agentic review:**

> Perform a final publish-readiness audit on this ACAP project. Check:
>
> 1. Does the README accurately describe the current application?
> 2. Are build instructions complete and correct?
> 3. Is a LICENSE file present with the correct license text?
> 4. Are there any TODO, FIXME, HACK, or XXX comments in the source?
> 5. Is there dead code (unreachable functions, commented-out blocks, unused variables)?
> 6. Are there any obsolete migration paths, temporary workarounds, or host-side sync helpers that should be removed?
> 7. Does `manifest.json` have accurate metadata (appName, vendor, version, friendlyName)?
> 8. Have all test devices and their IPs been removed from committed files?
> 9. Does `.gitignore` cover all generated and sensitive files?
> 10. Has the app been tested on the target device(s) and architecture(s)?
>
> Provide a go/no-go assessment with specific issues to fix before publishing.

**Manual checklist:**

- [ ] README is accurate and complete
- [ ] LICENSE file present (MIT unless otherwise justified)
- [ ] No TODO/FIXME/HACK comments remain
- [ ] No dead code or commented-out blocks
- [ ] No host-side workarounds if the app is fully on-camera
- [ ] Manifest metadata is accurate (no placeholders)
- [ ] No device IPs, usernames, or passwords in committed files
- [ ] `.gitignore` covers `.eap`, `.o`, `.env.devices`, `build/`
- [ ] Tested on real target device(s)
- [ ] Version number is correct and tagged in git

---

## Audit 6: Memory and Resource Safety

**Prompt for agentic review:**

> Review this ACAP project for memory safety and resource management issues. Check:
>
> 1. Is all dynamically allocated memory freed? Check every `malloc`/`calloc`/`g_malloc`/`g_strdup` for a corresponding `free`/`g_free`.
> 2. Are file handles closed after use? Check `fopen`/`fclose`, `open`/`close` pairs.
> 3. Are D-Bus connections properly cleaned up?
> 4. Are signal handlers safe? They should only set `volatile sig_atomic_t` flags — no complex logic, no library calls.
> 5. Is the main loop properly structured for clean shutdown on SIGTERM/SIGINT?
> 6. Are there any unbounded loops or allocations that could exhaust device memory?
> 7. If the app polls an API: is the poll interval bounded and configurable?
> 8. Are GLib main loop and event sources properly cleaned up?
>
> Flag any resource leak, unsafe signal handler, or unbounded resource consumption.

**Manual checklist:**

- [ ] All `malloc`/`g_malloc` paired with `free`/`g_free`
- [ ] All `fopen` paired with `fclose`
- [ ] Signal handlers only set atomic flags
- [ ] Clean shutdown on SIGTERM/SIGINT
- [ ] No unbounded memory growth (e.g., growing lists without limits)
- [ ] Poll intervals are bounded and configurable
- [ ] D-Bus and GLib resources cleaned up on exit

---

## Running a Full Audit

To run all audits as a single agentic prompt:

> Perform a complete security and quality audit of this ACAP project. Run the following checks in order:
>
> 1. **Network Exposure** — List all ports, verify proxy/FastCGI configuration, check access levels
> 2. **Credential Handling** — Search for hardcoded creds, verify D-Bus usage for local VAPIX, check git history
> 3. **Input Validation** — Trace all external input paths, check for injection (command, path, XSS, SQL), verify bounds checking
> 4. **Build/Packaging** — Verify Dockerfile, .dockerignore, manifest correctness, no stale artifacts
> 5. **Publish Readiness** — README accuracy, license, dead code, metadata, device IPs removed
> 6. **Memory/Resource Safety** — Leak detection, signal handler safety, clean shutdown, bounded resource use
>
> For each category, provide:
> - PASS / FAIL / WARN status
> - Specific file:line references for any findings
> - Recommended fix for each issue
>
> End with an overall go/no-go assessment.
