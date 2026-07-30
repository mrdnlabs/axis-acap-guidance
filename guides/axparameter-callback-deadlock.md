# AXParameter change callbacks: the reentrancy deadlock

Registering a change callback with `ax_parameter_register_callback()` is the correct way to make an
ACAP react to settings changes without a restart. But **reading a parameter from inside that
callback deadlocks the application**, and the failure is silent and confusing.

> **Empirical.** Observed on AXIS Q3538-SLVE, AXIS OS 12.11.77, ACAP Native SDK 12.11.0. Not
> documented by Axis; the reentrancy constraint is not mentioned in the Parameter API reference.

---

## Symptom

A `param.cgi?action=update` request **never returns**. At the same moment the ACAP stops answering
everything — its own web endpoints included — and nothing is written to the application log. From
the outside it looks like a device problem or a network problem, not an application bug:

```bash
# hangs forever, no response, no error
curl --anyauth -u "root:<password>" \
  "http://<device-ip>/axis-cgi/param.cgi?action=update&root.Myapp.SomeSetting=42"
```

The application does not crash, so `runMode: respawn` does not rescue it. Recovery needs an explicit
stop/start:

```bash
curl --anyauth -u "root:<password>" \
  "http://<device-ip>/axis-cgi/applications/control.cgi?action=stop&package=<appname>"
```

## Cause

`ax_parameter_get()` is a synchronous round trip. When called from inside a registered callback, it
waits for a reply on the same connection that is currently delivering that callback — which cannot
be processed until the callback returns. Both sides wait. The `param.cgi` request is caught in the
same cycle because it is blocked on the parameter write completing.

The trap is easy to fall into, because reloading settings is the obvious thing to do when settings
change:

```c
/* DEADLOCKS — do not do this */
static void on_param_changed(const gchar* name, const gchar* value, gpointer data) {
    reload_all_settings();      /* calls ax_parameter_get() internally */
}
```

## Fix: defer out of the callback

Do no parameter work inside the callback. Schedule it and return immediately, so the reload runs on
a clean stack:

```c
static guint pending_reload = 0;

static gboolean reload_on_idle(gpointer user_data) {
    pending_reload = 0;
    reload_all_settings();          /* safe here — not inside the callback */
    return G_SOURCE_REMOVE;
}

static void on_param_changed(const gchar* name, const gchar* value, gpointer data) {
    /* Coalesce: saving a settings form writes several parameters, and each one
     * fires its own callback. One reload covers them all. */
    if (pending_reload == 0)
        pending_reload = g_idle_add(reload_on_idle, NULL);
}
```

Remember to `g_source_remove()` the pending source during shutdown.

If you only need the one value that changed, the callback already receives `name` and `value` — using
those directly avoids any read at all, and is the cheapest fix of all.

## Related: don't hold a mutex across `ax_parameter_get()`

`ax_parameter_get()` is IPC, not a memory read. Holding an application lock across it means holding
that lock across I/O — which invites a second, unrelated deadlock once another thread (a metadata
callback, an HTTP handler) wants the same lock.

Read into a local copy first, then take the lock only to install the result:

```c
config_t fresh;

g_mutex_lock(&lock);
fresh = current_config;             /* start from what we have */
g_mutex_unlock(&lock);

reload_into(&fresh);                /* IPC, no lock held */

g_mutex_lock(&lock);
current_config = fresh;
g_mutex_unlock(&lock);
```

This is the same rule the [security audit](../standards/acap-security-audit.md) applies to
concurrency generally: no lock held across I/O.

## Verifying the fix

The regression test is the exact call that hung. It must return promptly, the new value must be
visible to the app, and the app must stay responsive:

```bash
curl --anyauth -u "root:<password>" \
  "http://<device-ip>/axis-cgi/param.cgi?action=update&root.Myapp.SomeSetting=42"
# then confirm the app still answers, and picked the value up
curl --anyauth -u "root:<password>" "http://<device-ip>/local/<appname>/api/health"
```

Note the group name is `root.<Appname>.*`, whose capitalization may not match your manifest
`appName` — see [acap-manifest-gotchas.md](./acap-manifest-gotchas.md#parameter-group-capitalization-is-unpredictable).

---

## References

- [Parameter API (AXParameter)](https://developer.axis.com/acap/api/native-sdk-api/#parameter-api) —
  `ax_parameter_register_callback()`, `ax_parameter_get()`, `ax_parameter_set()`. The reentrancy
  constraint described above is **not** stated here; the deadlock is an observed behavior.
- [`axparameter` SDK example](https://github.com/AxisCommunications/acap-native-sdk-examples/tree/main/axparameter) —
  official callback registration example.
- [VAPIX — parameter management](https://developer.axis.com/vapix/network-video/parameter-management/) —
  `param.cgi?action=update`, the external path that hangs alongside the app.
- [GLib main loop](https://docs.gtk.org/glib/main-loop.html) — `g_idle_add()` / `g_source_remove()`
  used for the deferral.
