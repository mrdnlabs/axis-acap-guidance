# libcurl in an ACAP: hardening checklist

An ACAP that makes outbound HTTP calls (webhooks, cross-camera VAPIX, MQTT-over-HTTP bridges, alarm dispatch to a door controller) will almost always use libcurl — it ships in the SDK sysroot and works cleanly in the `axisecp/acap-native-sdk` image. Getting `curl_easy_init` + `curl_easy_perform` running is easy. Getting it running _safely_ under the ACAP threading model is a checklist most first drafts miss.

This guide is the checklist. Everything here comes from real bugs found on an anti-tailgating ACAP that dispatches alarms over libcurl from detached pthreads.

---

## The three failure modes to design against

1. **libcurl kills the process via a signal.** Default libcurl uses `SIGALRM` for DNS timeouts. In a multithreaded ACAP the signal is delivered to an unpredictable thread. On top of that, a half-closed TCP connection raises `SIGPIPE` on write. Either one, unhandled, terminates the whole ACAP.
2. **The scheme allowlist is wide open.** libcurl by default accepts `file://`, `gopher://`, `dict://`, `sftp://`, `smb://`, whatever it was built with. Any user-configurable URL is a potential SSRF or local-file-read primitive.
3. **Credentials leak.** Basic auth over HTTP is cleartext. Digest auth on a redirect can be forwarded to the redirect target. `syslog` calls that log the request URL can capture `user:pass@…` userinfo.

The rest of this guide is the concrete `curl_easy_setopt` incantations that close each of them.

---

## The per-handle setopt block

Set every one of these on every `curl_easy_init` handle. Comments explain why.

```c
/* Multithread safety.  Without this, libcurl uses SIGALRM for DNS
 * timeouts and the signal is delivered to an unpredictable thread.
 * In a detached-worker model that typically means the main loop. */
curl_easy_setopt(curl, CURLOPT_NOSIGNAL, 1L);

/* TLS is on-by-default on modern libcurl, but say so explicitly.
 * A future toolchain that flips the default is a silent regression. */
curl_easy_setopt(curl, CURLOPT_SSL_VERIFYPEER, 1L);
curl_easy_setopt(curl, CURLOPT_SSL_VERIFYHOST, 2L);

/* Scheme allowlist.  libcurl accepts every scheme it was built with;
 * a user-set URL like file:///etc/passwd otherwise works.  Prefer
 * CURLOPT_PROTOCOLS_STR (libcurl 7.85+); fall back to the bitmask on
 * older sysroots. */
#ifdef CURLOPT_PROTOCOLS_STR
curl_easy_setopt(curl, CURLOPT_PROTOCOLS_STR,       "http,https");
curl_easy_setopt(curl, CURLOPT_REDIR_PROTOCOLS_STR, "http,https");
#else
curl_easy_setopt(curl, CURLOPT_PROTOCOLS,
                 (long)(CURLPROTO_HTTP | CURLPROTO_HTTPS));
curl_easy_setopt(curl, CURLOPT_REDIR_PROTOCOLS,
                 (long)(CURLPROTO_HTTP | CURLPROTO_HTTPS));
#endif

/* Bound redirects.  Default is unlimited.  Also drop auth on redirect
 * so digest/basic credentials do not leak to a redirect target. */
curl_easy_setopt(curl, CURLOPT_MAXREDIRS,          3L);
curl_easy_setopt(curl, CURLOPT_UNRESTRICTED_AUTH,  0L);

/* Timeouts.  Without both, a half-open target hangs the worker (and
 * blocks its slot in the drain path at shutdown, see the lifecycle
 * guide).  15s / 10s is a defensible LAN-plus-a-bit default. */
curl_easy_setopt(curl, CURLOPT_CONNECTTIMEOUT,     10L);
curl_easy_setopt(curl, CURLOPT_TIMEOUT,            15L);

/* Identify yourself.  Default UA leaks the libcurl version and does
 * not attribute traffic to your ACAP on the receiving end. */
curl_easy_setopt(curl, CURLOPT_USERAGENT, "myacap/1.0.0");

/* Auth: prefer Digest but accept Basic.  AXIS OS 12.9 with a Basic-
 * only policy will 401 on Digest-only; libcurl picks the strongest
 * offered by the server.  Explicitly exclude NTLM. */
if (username[0]) {
    curl_easy_setopt(curl, CURLOPT_USERPWD, userpwd);
    curl_easy_setopt(curl, CURLOPT_HTTPAUTH,
                     (long)(CURLAUTH_DIGEST | CURLAUTH_BASIC));
}
```

---

## And in `main`

```c
/* One unsignaled SIGPIPE from libcurl on a half-closed connection
 * would terminate the whole ACAP. */
signal(SIGPIPE, SIG_IGN);
```

Set it before any thread that will use libcurl is spawned. Top of `main` after `openlog` is the right place. `CURLOPT_NOSIGNAL` and `SIG_IGN` are complementary: `NOSIGNAL` stops libcurl from _raising_ `SIGALRM` on DNS timeout; `SIG_IGN` on `SIGPIPE` stops the kernel from delivering it on a broken write. You need both.

---

## Default outbound scheme: https, with an explicit opt-out

Most ACAPs that talk to another Axis device on the LAN send credentials — either digest challenge/response or basic auth. Even digest can be a downgrade target. Default your outbound scheme to `https://` and expose a config field (call it `MyAppInsecure` or similar, `type: bool:false,true`, default `false`) that lets an operator opt in to `http://` for a legacy target that lacks TLS.

The Axis-CGI action-URL pattern from the sample anti-tailgating ACAP:

```c
char *insecure_str = config_get_string("MyAppInsecure", "false");
bool insecure = (insecure_str && strcmp(insecure_str, "true") == 0);
free(insecure_str);
const char *scheme = insecure ? "http" : "https";

snprintf(url, sizeof(url),
         "%s://%s/axis-cgi/io/port.cgi?action=%s%%3A%%2F",
         scheme, host, port);
```

For a user-supplied full URL, validate it before dispatch:

```c
if (strncmp(url, "http://", 7) != 0 && strncmp(url, "https://", 8) != 0) {
    /* Refuse: file://, gopher://, sftp://, ... */
    return dispatch_failed;
}
if (!insecure && strncmp(url, "http://", 7) == 0) {
    /* Belt-and-suspenders with the manifest opt-in. */
    return dispatch_failed;
}
```

The scheme allowlist inside `do_curl_request` (above) is your last line of defence; the config-POST validator and the dispatch-time check catch it earlier with a clearer error message.

---

## Do not log the full URL

Any admin-set URL can contain userinfo (`https://user:pass@host/path`) or a query token (`?token=SECRET`). If your dispatch logs the URL, both end up in syslog. A tiny sanitizer:

```c
/* Copy scheme://host[:port]/path.  Drop userinfo, query, fragment. */
static void sanitize_url_for_log(const char *in, char *out, size_t out_len)
{
    if (!in || !out || out_len == 0) return;
    out[0] = '\0';
    const char *scheme_end = strstr(in, "://");
    if (!scheme_end) { snprintf(out, out_len, "%s", in); return; }
    const char *authority = scheme_end + 3;
    const char *at    = strchr(authority, '@');
    const char *slash = strchr(authority, '/');
    if (at && (!slash || at < slash)) authority = at + 1;

    size_t scheme_len = (size_t)(scheme_end - in);
    if (scheme_len + 3 >= out_len) { snprintf(out, out_len, "<url>"); return; }
    memcpy(out, in, scheme_len);
    memcpy(out + scheme_len, "://", 3);
    size_t j = scheme_len + 3;
    for (size_t i = 0; authority[i] &&
                       authority[i] != '?' && authority[i] != '#' &&
                       j + 1 < out_len; i++)
        out[j++] = authority[i];
    out[j] = '\0';
}
```

Then `syslog(LOG_INFO, "outbound: %s %s", method, safe_url)` instead of `%s %s`, `method, url`.

---

## Zero credentials before free

libcurl copies `CURLOPT_USERPWD` into its own buffer, but any credential-holding struct in your code should be zeroed before `free`. glibc has `explicit_bzero` for exactly this — the compiler cannot elide it, so plaintext does not linger in the heap where a core dump could recover it.

```c
static void free_args_zeroed(MyArgs *a)
{
    if (!a) return;
    explicit_bzero(a->user,   sizeof(a->user));
    explicit_bzero(a->pass,   sizeof(a->pass));
    explicit_bzero(a->header, sizeof(a->header));
    free(a);
}
```

Apply on every free path (success, dispatch failure, `pthread_create` failure).

---

## What this does NOT solve

- **On-flash residue.** If you store the credential in `ax_parameter` (even with `password:maxlen=N`), the value is unencrypted on flash. See [standards §4.3](../standards/acap-development-standards.md#43-credential-storage) — no Axis keystore exists.
- **Compromised target.** Nothing here stops a legitimate credential going to a legitimate URL that has been compromised. TLS certificate pinning (`CURLOPT_PINNEDPUBLICKEY`) is the next step if you know the target's cert in advance.
- **Certificate revocation.** libcurl on Axis honours the system CA bundle but does not check CRL/OCSP by default. If your threat model needs it, wire in `CURLOPT_SSL_CTX_FUNCTION` and drive OpenSSL manually.

---

## Checklist to run against your own code

- [ ] `signal(SIGPIPE, SIG_IGN)` at top of `main`
- [ ] `CURLOPT_NOSIGNAL = 1L` on every handle
- [ ] `CURLOPT_SSL_VERIFYPEER = 1L` and `CURLOPT_SSL_VERIFYHOST = 2L` explicit
- [ ] `CURLOPT_PROTOCOLS_STR` (or the bitmask) restricts to `http,https`
- [ ] `CURLOPT_REDIR_PROTOCOLS_STR` matches
- [ ] `CURLOPT_MAXREDIRS` set (typically 3)
- [ ] `CURLOPT_UNRESTRICTED_AUTH = 0L`
- [ ] `CURLOPT_CONNECTTIMEOUT` + `CURLOPT_TIMEOUT` set to bounded values
- [ ] `CURLOPT_USERAGENT` identifies your ACAP
- [ ] Default outbound scheme is `https://`; opt-in flag exposes `http://` for legacy targets
- [ ] User-supplied URLs are validated for `http[s]://` prefix at config-POST time
- [ ] The URL passed to `syslog` is sanitized (userinfo + query stripped)
- [ ] Every credential-holding struct is zeroed before `free`
- [ ] Detached-thread cleanup drains in-flight workers before `curl_global_cleanup` (see [acap-lifecycle-and-cleanup.md](./acap-lifecycle-and-cleanup.md))

---

## References

- [libcurl `curl_easy_setopt`](https://curl.se/libcurl/c/curl_easy_setopt.html) — every option above with the official semantics
- [libcurl `CURLOPT_PROTOCOLS_STR`](https://curl.se/libcurl/c/CURLOPT_PROTOCOLS_STR.html) — the newer API; added in libcurl 7.85 (2022)
- [libcurl `CURLOPT_NOSIGNAL`](https://curl.se/libcurl/c/CURLOPT_NOSIGNAL.html) — official warning: *"must be set to 1L when using libcurl in a multi-threaded environment."*
- [standards §4.3 Credential Storage](../standards/acap-development-standards.md#43-credential-storage) — where the outbound credential itself should live
- Empirical: anti-tailgating ACAP commits `d56c738` (Phase B) — a diff of every finding above applied at once
