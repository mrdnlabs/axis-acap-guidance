# ACAP Security Audit Prompts and Checklists

> **What:** A structured set of audit prompts and manual checklists for reviewing ACAP application code for security vulnerabilities, packaging issues, and publish readiness. Each audit can be used as a standalone checklist or fed directly to an agentic code review tool.
>
> **Who:** Developers, reviewers, or agentic tools performing a security and quality review of an ACAP project.
>
> **When:** Before publishing or deploying an ACAP to production devices. Can also be run periodically during development to catch issues early. The "Full Audit" prompt at the bottom runs all seven checks in sequence.
>
> **Before recording any finding, read [Common false positives](#common-false-positives).** Repeat audits — and agentic reviews in particular — reliably produce confident, plausible, wrong findings; verifying against `HEAD` first is what separates a useful audit from noise.
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
- [ ] No `"anonymous"` access unless explicitly justified and documented (and note: `anonymous` is valid only on `reverseProxy` entries, not on `httpConfig`/FastCGI)
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
> 5. Are credentials ever logged (syslog, printf, debug output)? Include **request bodies** — a settings
>    POST can carry a cleartext password, so logging the raw body leaks it. Log a byte count instead.
> 6. Are credentials cleared from memory when no longer needed?
> 7. Is `.env.devices` or equivalent in `.gitignore`?
> 8. Check git history for any previously committed credentials.
> 9. If the app persists credentials to a file it writes itself (not `ax_parameter`), what mode is that
>    file created with? Anything world- or group-readable exposes the secret to other processes and to
>    device/config backups.
> 10. For **outbound** credentialed requests: can a redirect or a configurable hostname re-target them?
>     Following a redirect can send credentials to an attacker-chosen host; an unvalidated host field is
>     an SSRF primitive.
>
> Flag any credential that is hardcoded, stored in plaintext parameters, logged, or committed to version control.

**Manual checklist:**

- [ ] No hardcoded credentials in source code
- [ ] No passwords in `manifest.json` default parameter values
- [ ] Local VAPIX uses D-Bus `GetCredentials` + loopback, not stored credentials
- [ ] Remote-device credentials use `manifest.json` `"type": "password:maxlen=N"` (NOT `"String"` and NOT `"hidden:string"` — `hidden` does not mask `param.cgi`)
- [ ] `curl https://<device>/axis-cgi/param.cgi?action=list&group=root.<App>` returns `******` for every credential field, not the value
- [ ] API does not echo the password back (use a `…Configured` boolean instead) and provides an explicit clear path
- [ ] Credentials never written to syslog or debug output — **including request bodies** (log a byte count, not the body)
- [ ] Config files the app writes itself are created with restrictive permissions (e.g. `fchmod(fileno(f), S_IRUSR | S_IWUSR)` → `0600`), not just the default umask
- [ ] Credentialed outbound requests do **not** follow redirects (leave `CURLOPT_FOLLOWLOCATION` unset) — a redirect can re-target the request, and its credentials, to another host
- [ ] A configurable remote host is validated before URL construction — reject control characters and `/ \ @ ? # %`, which are the characters that let a "host" rewrite the path, query, or authority (SSRF / credential redirection)
- [ ] `.env.devices` and similar files are in `.gitignore`
- [ ] `git log -p --all -S 'password'` and `git log -p --all -S 'secret'` return no real credentials
- [ ] HTTPS used for all remote device communication
- [ ] If TLS peer verification is disabled (common with self-signed certs on peripherals), that is a **documented, deliberate** trust-boundary decision with an opt-in path to enable it — not an unexamined default

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
- [ ] VAPIX URL parameters are properly escaped (e.g. `curl_easy_escape()` on any value interpolated into a URL)
- [ ] **GET is side-effect-free**; every state-changing action requires POST (return `405` otherwise). A GET that mutates state can be triggered by a prefetch, a crawler, or a cross-site `<img>` tag
- [ ] Malformed JSON bodies are rejected with `400`, not dereferenced (check every `cJSON_Parse()` result for NULL)

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
- [ ] Signal handlers only set atomic flags — **or** use `g_unix_signal_source_new()`, which dispatches on the main loop where calling `g_main_loop_quit()` is safe (preferred in a GLib app; it sidesteps async-signal-safety entirely)
- [ ] Clean shutdown on SIGTERM/SIGINT
- [ ] No unbounded memory growth (e.g., growing lists without limits)
- [ ] Poll intervals are bounded and configurable
- [ ] D-Bus and GLib resources cleaned up on exit

---

## Audit 7: Concurrency and Thread Safety

> **Why this is its own audit:** an ACAP is multi-threaded *even if you never spawn a thread*. The GLib
> main loop, FastCGI request handlers (commonly a dedicated thread), `GThreadPool` workers, and SDK
> callbacks (larod job callbacks, VDO, D-Bus, MQTT delivery) can all touch your state concurrently.
> Concurrency bugs are the ones least likely to appear in testing and most likely to appear in the
> field, so they warrant a deliberate pass rather than a line item under memory safety.

**Prompt for agentic review:**

> Review this ACAP project for concurrency and thread-safety issues. Check:
>
> 1. **Enumerate the execution contexts first.** List every context that can run this app's code: the
>    GLib main loop, HTTP/FastCGI handler thread(s), each `GThreadPool` (and its `max_threads`), SDK
>    async callbacks (larod job completion, VDO frame delivery, MQTT), and timer/idle sources. Do this
>    before judging any individual variable — most false findings come from guessing this wrong.
> 2. For each piece of shared mutable state, which contexts read it and which write it? Is every access
>    serialized by the same lock? Name the owning lock for each.
> 3. Is any lock held across a blocking call — network/curl, file I/O, or a blocking SDK call? On the
>    main loop this stalls frame processing; generally it invites deadlock and latency spikes.
> 4. Are there lock-ordering inversions — two locks acquired in opposite orders on different paths?
> 5. Is every lock released on **all** early-return and error paths?
> 6. Does any non-main-loop context mutate configuration/settings while the main loop reads it? Config
>    trees are a classic use-after-free: one thread frees a node while another walks it. Prefer
>    marshalling mutations to a single owning context.
> 7. Do async SDK callbacks reference buffers that could be freed or reused while the job is in flight?
> 8. Can a synchronization primitive be destroyed while a late callback might still post to it (e.g.
>    destroying a semaphore at cleanup while an SDK poll thread still holds a reference)?
> 9. Are worker queues bounded? An unbounded queue behind a slow device turns a backlog into
>    ever-increasing lag or memory growth.
> 10. Is shutdown ordered so in-flight jobs drain **before** the libraries they use are torn down (e.g.
>     drain pools before `curl_global_cleanup()`)?
>
> For each finding, state the specific interleaving that triggers it. If you cannot describe the
> interleaving, it is probably not a real race — see "Common false positives" below.

**Manual checklist:**

- [ ] The set of execution contexts is written down (main loop, HTTP thread, pools + `max_threads`, SDK callbacks)
- [ ] Every shared mutable variable has one documented owning lock
- [ ] No lock is held across network I/O, file I/O, or a blocking SDK call
- [ ] Consistent lock ordering; no inversions
- [ ] Locks released on every error/early-return path
- [ ] Configuration is mutated from exactly one context (marshal to the main loop if needed)
- [ ] Buffers passed to async callbacks outlive the job
- [ ] Sync primitives are not destroyed while a late callback could still use them
- [ ] Worker queues are bounded, with a defined drop/coalesce policy
- [ ] Shutdown drains in-flight work before library teardown

---

## Common false positives

Concurrency and C reviews — especially agentic ones — reliably produce a handful of confident,
plausible, and **wrong** findings. Check these before recording anything.

| Reported claim | Why it is usually wrong |
|---|---|
| *"This mutex-protected variable needs `volatile`."* | `volatile` is **not** a threading primitive in C. It provides no atomicity and no inter-thread ordering. A mutex lock/unlock is already both a compiler barrier and a memory barrier, so data accessed consistently under a lock does **not** need `volatile`. Its legitimate uses are MMIO and `sig_atomic_t` flags. Adding `volatile` to "fix" a race hides the real question — which is whether the locking is consistent. |
| *"Race between the check and the use in this worker."* | Check the pool's `max_threads` first. A `GThreadPool` created with `max_threads = 1` serializes every job, so jobs cannot race each other by construction. A "race" that requires two workers is not a finding on a single-threaded pool. |
| *Findings restated from a previous audit report.* | A reviewer handed an earlier audit document will re-report its findings as if current — including ones already fixed. **Re-verify every finding against `HEAD` before recording it.** This is the single most common failure mode of a repeat audit. |
| *"Dead code / an unreachable branch is still present."* | Verify by reading the file at `HEAD`, not by trusting a changelog, an old review, or a memory of the codebase. |
| *"Missing free / leak"* on a path that cannot execute. | Distinguish *"leaks on a path that runs"* from *"leaks only if an allocation that never fails, fails."* Both may be worth fixing, but only the first is a real defect; label them accordingly. |
| *"Tested on device"* treated as proof it works. | Installing and starting is not working — see [dlpu-memory-contention.md](../guides/dlpu-memory-contention.md). Verify the app's own health endpoint, not just `list.cgi`'s `Status="Running"`. |

**Rule of thumb:** if you cannot state the concrete interleaving (or concrete input) that produces the
failure, do not file it as a defect. Record it as a question instead. A precise "no finding here, and
here is why" is more valuable to the next reviewer than a speculative warning.

---

## Running a Full Audit

To run all audits as a single agentic prompt:

> Perform a complete security and quality audit of this ACAP project. Run the following checks in order:
>
> 1. **Network Exposure** — List all ports, verify proxy/FastCGI configuration, check access levels
> 2. **Credential Handling** — Search for hardcoded creds, verify D-Bus usage for local VAPIX, check file modes, redirect/SSRF on outbound calls, check git history
> 3. **Input Validation** — Trace all external input paths, check for injection (command, path, XSS, SQL), verify bounds checking, GET/POST contract
> 4. **Build/Packaging** — Verify Dockerfile, .dockerignore, manifest correctness, no stale artifacts
> 5. **Publish Readiness** — README accuracy, license, dead code, metadata, device IPs removed
> 6. **Memory/Resource Safety** — Leak detection, signal handler safety, clean shutdown, bounded resource use
> 7. **Concurrency/Thread Safety** — Enumerate execution contexts first, then lock discipline, locks across I/O, async callback lifetimes, shutdown ordering
>
> For each category, provide:
> - PASS / FAIL / WARN status
> - Specific file:line references for any findings
> - Recommended fix for each issue
>
> **Verify before you report.** Read the code at `HEAD` for every finding — do not restate findings from
> an earlier audit document, and do not report a race you cannot describe an interleaving for. Check your
> findings against "Common false positives" above and drop the ones that match. A short, verified list
> beats a long, speculative one; recording *"checked X, not a defect, because Y"* is a useful result.
>
> End with an overall go/no-go assessment.

---

## References

The audit rules above map to these official sources:

- [Manifest schema v2.0.0 — field descriptions](https://developer.axis.com/acap/reference/manifest-schemas/manifest-v2/schema-field-descriptions-v2.0.0/) — `reverseProxy` / `httpConfig` `access` levels (Audit 1; `anonymous` is `reverseProxy`-only).
- [VAPIX access for ACAP applications](https://developer.axis.com/acap/develop/VAPIX-access-for-ACAP-applications/) — D-Bus `GetCredentials` + `127.0.0.12` for local VAPIX (Audit 2).
- [ACAP 12.7 release notes](https://developer.axis.com/acap/release-notes/12.7/) — `password` / `writeonly` parameter masking in the audit log (Audit 2).
- [AXIS OS Hardening Guide](https://help.axis.com/en-us/axis-os-hardening-guide) — device security baseline (general).
- [libcurl — `CURLOPT_FOLLOWLOCATION`](https://curl.se/libcurl/c/CURLOPT_FOLLOWLOCATION.html) and [`CURLOPT_UNRESTRICTED_AUTH`](https://curl.se/libcurl/c/CURLOPT_UNRESTRICTED_AUTH.html) — redirect handling and when credentials are sent to a redirected host (Audit 2).
- [libcurl — `curl_easy_escape`](https://curl.se/libcurl/c/curl_easy_escape.html) — URL-encoding values interpolated into a URL (Audit 3).
- [GLib — `g_unix_signal_source_new`](https://docs.gtk.org/glib/type_func.Source.unix_signal_new.html) — dispatch a signal on the main loop instead of an async-signal-safe handler (Audit 6).
- [GLib — Threads](https://docs.gtk.org/glib/threads.html) and [`GThreadPool`](https://docs.gtk.org/glib/struct.ThreadPool.html) — `GMutex` semantics and `max_threads` (Audit 7; a pool with `max_threads = 1` serializes its jobs).
- [ISO C11 §5.1.2.4 / cppreference — `volatile`](https://en.cppreference.com/w/c/language/volatile) and [memory order](https://en.cppreference.com/w/c/atomic/memory_order) — `volatile` provides neither atomicity nor inter-thread ordering; it is not a substitute for a mutex or an atomic (Audit 7 / false positives).
