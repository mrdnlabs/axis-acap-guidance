# Enabling unsigned ACAP apps programmatically

On AXIS OS 12.x, unsigned apps are disabled by default. The camera can be configured over VAPIX with the ACAP application configuration CGI.

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

## Notes

- This is the supported programmatic path; no GUI interaction is required.
- On AXIS OS 12.0 and later, the default is `false`.
- After enabling it, unsigned `.eap` uploads via `/axis-cgi/applications/upload.cgi` will install successfully.
