# Axis HTTP Authentication: Basic vs Digest (and why forcing Digest can 401)

If your client (an ACAP calling another device, a script, or libcurl code) **forces** Digest authentication and gets a `401 Unauthorized` — while Basic succeeds — the cause is almost never the specific CGI or the device model. It's the device's configurable **HTTP authentication policy** combined with the protocol (HTTP vs HTTPS) you're using.

## The setting: `Network.HTTP.AuthenticationPolicy`

Axis devices expose a single VAPIX parameter that decides which authentication scheme the HTTP, HTTPS, and RTSP servers offer. Available since AXIS OS 5.70; admin read/write.

| Policy | HTTP | HTTPS | RTSP |
|--------|------|-------|------|
| `Basic` | Basic | Basic | Basic |
| `Digest` | Digest | Digest | Digest |
| `Basic_Digest` | Digest | Digest | Basic & Digest |
| **`Recommended`** (default) | **Digest** | **Basic** | Digest |

Read or set it via `param.cgi`:

```bash
# Read
curl --anyauth -u root:<password> \
  "https://<device-ip>/axis-cgi/param.cgi?action=list&group=Network.HTTP.AuthenticationPolicy"

# Set (e.g. force Digest everywhere)
curl --anyauth -u root:<password> \
  "https://<device-ip>/axis-cgi/param.cgi?action=update&Network.HTTP.AuthenticationPolicy=Digest"
```

## The gotcha

Look at the **`Recommended`** (default) row: **HTTPS offers Basic, HTTP offers Digest.** This is counterintuitive — you'd expect HTTPS to use the stronger scheme. The logic is the reverse: HTTPS already encrypts the channel, so Basic is considered acceptable there; plain HTTP has no encryption, so Digest is used to avoid sending the password in the clear.

The practical consequence:

> A client that talks to a device **over HTTPS** and **forces Digest** will get `401` under the default policy — because the server only offered Basic on HTTPS.

This was originally observed driving a C1110-E peripheral over HTTPS (its `siren_and_light.cgi`, `mediaclip.cgi`, `param.cgi` all "Digest-401'd, Basic-200'd"), but it is **not** a property of those CGIs or that model — any Axis device on the default policy behaves this way over HTTPS. Change the policy to `Digest`, or call over HTTP, and the same endpoints accept Digest.

## The fix: negotiate, don't force

Don't hard-code a single auth scheme in your client. Let the server's `WWW-Authenticate` header decide:

```c
/* libcurl: accept whatever the device offers (Basic or Digest) */
curl_easy_setopt(curl, CURLOPT_HTTPAUTH, CURLAUTH_ANY);   /* not CURLAUTH_DIGEST */
```

```bash
# curl CLI
curl --anyauth -u root:<password> "https://<device-ip>/axis-cgi/..."
```

`CURLAUTH_ANY` / `--anyauth` issues an unauthenticated request first, reads the offered scheme, and responds correctly. Forcing `CURLAUTH_DIGEST` (or `--digest`) skips that negotiation and fails whenever the server only offers Basic.

## Security implication when Basic is in play

If you can't force Digest (e.g. you're calling over HTTPS under the default policy, or hitting a device whose policy is `Basic`), the password travels **Basic-encoded — base64, effectively cleartext.** Its confidentiality then rests **entirely on TLS**. Two things follow:

- **Axis devices ship self-signed certificates.** A client that disables certificate verification (libcurl `CURLOPT_SSL_VERIFYPEER=0` / `VERIFYHOST=0`) is implicitly trusting the LAN segment as its security boundary — there's no protection against an on-path attacker. Document this posture in your README.
- **Offer a verify option.** For deployments where the peer presents a CA-trusted (or pinned) certificate, expose an opt-in `tls_verify` config that sets `VERIFYPEER=1`/`VERIFYHOST=2`. Only then is Basic-over-TLS actually protected against an on-path attacker.
- **Don't follow redirects on credentialed one-shot calls.** A play/start/config CGI has no legitimate redirect; leaving `CURLOPT_FOLLOWLOCATION` off avoids replaying credentials to an unexpected host.

## Quick checklist

- Getting `401` only when forcing Digest? Check `Network.HTTP.AuthenticationPolicy` and whether you're on HTTP vs HTTPS.
- Use `CURLAUTH_ANY` / `--anyauth` in clients instead of pinning a scheme.
- When the negotiated scheme is Basic, treat TLS as the only thing protecting the password — verify certs where you can, and document the trust boundary where you can't.

## Reference

- [Network settings — VAPIX](https://developer.axis.com/vapix/network-video/network-settings/) (`Network.HTTP.AuthenticationPolicy`)
- [VAPIX authentication](https://developer.axis.com/vapix/authentication/)
- [AXIS OS hardening guide](https://help.axis.com/en-us/axis-os-hardening-guide)
