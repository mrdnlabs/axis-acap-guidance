# Enabling unsigned ACAP apps programmatically

From AXIS OS 12.0, signing of ACAP applications is required by default (unsigned apps are disabled), but it can be toggled off over VAPIX with the ACAP application configuration CGI. The XML replies shown below are representative of the documented `<reply result="ok">…</reply>` shape — confirm the exact body on your firmware.

## Read current setting

```bash
curl --request POST \
  --anyauth \
  --user "root:<password>" \
  "http://<camera-ip>/axis-cgi/applications/config.cgi?action=get&name=AllowUnsigned"
```

Expected success response:

```xml
<reply result="ok">
  <param name="AllowUnsigned" value="false"/>
</reply>
```

## Enable unsigned apps

```bash
curl --request POST \
  --anyauth \
  --user "root:<password>" \
  "http://<camera-ip>/axis-cgi/applications/config.cgi?action=set&name=AllowUnsigned&value=true"
```

Expected success response:

```xml
<reply result="ok">
  <param name="AllowUnsigned" value="true"/>
</reply>
```

## Symptom: `upload.cgi` answers `Error:2`

If `AllowUnsigned` is `false`, uploading an unsigned `.eap` fails with a bare, unexplained:

```
Error:2
```

There is no message naming signing as the cause, so it is easy to misread as a corrupt package,
a bad path, or an auth problem. **Check `AllowUnsigned` first** (the `action=get` call above) before
debugging anything else — if authentication were the issue you would get a 401, not `Error:2`.

**A factory reset re-arms this.** A device that accepted your unsigned build for months will reject
it after a reset, because the toggle returns to its `false` default. If an upload that used to work
suddenly returns `Error:2`, suspect a reset (or a firmware upgrade) rather than your package. A reset
also restarts the bundled analytics, which can separately break a DLPU app — see
[dlpu-memory-contention.md](./dlpu-memory-contention.md).

## Notes

- This is the supported programmatic path; no GUI interaction is required.
- On AXIS OS 12.0 and later, the default is `false`.
- After enabling it, unsigned `.eap` uploads via `/axis-cgi/applications/upload.cgi` will install successfully.
- Installing successfully is **not** the same as working — verify the app's own health/status
  endpoint afterwards, not just `list.cgi`'s `Status="Running"`.
- **Not future-proof.** Axis has signaled that signing may become mandatory (no toggle) in a future major release. Don't bake an "enable unsigned" step into long-lived automation — sign your packages instead where you can.

---

## References

- [Accept or deny unsigned ACAP applications](https://developer.axis.com/acap/3/services-for-partners/accept-or-deny-unsigned-acap-applications/) — `config.cgi?action=get|set&name=AllowUnsigned&value=true|false`.
- [VAPIX — Application API](https://developer.axis.com/vapix/applications/application-api/) — `/axis-cgi/applications/upload.cgi` for uploading `.eap` packages, and the `AllowUnsigned` parameter.
- [ACAP 12.0 release notes](https://developer.axis.com/acap/release-notes/12.0/) — "From AXIS OS 12.0, signing of ACAP applications will be required by default, but can still be disabled with a toggle."
