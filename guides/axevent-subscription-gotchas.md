# AXEvent subscription gotchas

Subscribing to device events with the AXEvent API is easy to get *subtly*
wrong in ways that pass a quick demo and then misbehave in the field: the
wrong key name silently disables a filter, a stateful topic fires once on
subscribe, and a settings UI that resubscribes on every save drops events.

> **Empirical.** Verified on an AXIS Q3538-SLVE, AXIS OS 12.11.77, ACAP
> Native SDK 12.9/12.11, against the device's own `GetEventInstances`
> declaration.

---

## Read the device's own declaration first

Do not guess topic paths or key names. Ask the device what it actually
declares and copy the exact namespaces, source keys, and data keys:

```bash
curl -k --anyauth -u "$U:$P" \
  -H 'Content-Type: application/soap+xml' \
  -d '<?xml version="1.0"?><soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope" xmlns:aev="http://www.axis.com/vapix/ws/event1"><soap:Body><aev:GetEventInstances/></soap:Body></soap:Envelope>' \
  "https://$CAM/vapix/services"
```

Each `wstop:topic="true"` node is a topic; `<aev:SimpleItemInstance>` entries
are its keys. A key with `isPropertyState="true"` is the **data** key (the
state); the others are **source** keys (which instance). The `aev:NiceName`
on a source value tells you the human label vs. the wire value — this is
where off-by-one bugs hide.

---

## Digital input: right topic, right key, 0-based port

A digital-input ACAP commonly subscribes to the ONVIF
`tns1:Device/Trigger/DigitalInput` topic. On AXIS OS that topic carries a
**string** source key `InputToken` and a boolean `LogicalState`. If you
instead filter on an integer key named `port`, the lookup silently fails,
**the filter is skipped, and every input port fires your handler** — a
door-contact input can then authorize whatever the event was meant to gate.

Prefer the Axis-native port topic, whose declaration is unambiguous:

```
topic0 = tns1:Device
topic1 = tnsaxis:IO
topic2 = tnsaxis:Port
  source key "port"  : xsd:int   — 0-based!  value 0 is labelled "Port 1",
                                   value -1 is "Any"
  data   key "state" : xsd:boolean (property state)
```

```c
AXEventKeyValueSet *kv = ax_event_key_value_set_new();
ax_event_key_value_set_add_key_value(kv, "topic0", "tns1",    "Device", AX_VALUE_TYPE_STRING, NULL);
ax_event_key_value_set_add_key_value(kv, "topic1", "tnsaxis", "IO",     AX_VALUE_TYPE_STRING, NULL);
ax_event_key_value_set_add_key_value(kv, "topic2", "tnsaxis", "Port",   AX_VALUE_TYPE_STRING, NULL);
gint port = 0;  /* device-native 0-based index; "Port 1" in the UI */
ax_event_key_value_set_add_key_value(kv, "port", NULL, &port, AX_VALUE_TYPE_INT, NULL);
```

Two consequences for your config model:

- **Store the device-native 0-based index**, not a 1-based "Port N". If you
  must show "Port 1" to the operator, map `index -> index+1` in the UI only,
  and keep the stored value equal to the event's `port` value. Mixing the two
  is a latent off-by-one that subscribes to the wrong port.
- **`"0"` is a valid port.** A `disabled` sentinel must be `"none"`/empty, not
  `"0"` — otherwise port 0 ("Port 1") can never be selected.

`root.IOPort.I<n>.*` parameters are also 0-based, so they line up with the
event's `port` value directly.

---

## Stateful topics redeliver on subscribe — baseline it

A property (stateful) topic delivers its **current** value immediately after
you subscribe. If your handler treats every `active=true` / `state=true` as a
fresh edge, enabling the app (or changing a setting that resubscribes) while
someone is mid-zone or an input is held high will spuriously fire — consume a
token, raise an alarm, mint a token.

Swallow the first delivery after each (re)subscribe as a baseline:

```c
static gboolean baseline_pending = FALSE;   /* set TRUE in do_subscribe() */

static void on_event(guint sub, AXEvent *ev, gpointer d) {
    /* ... read the state ... */
    if (baseline_pending) { baseline_pending = FALSE; ax_event_free(ev); return; }
    /* act on the real edge */
}
```

Counting topics (e.g. AOA `crosslinecounting`) have no `active`/`state` key
and fire once per event, so don't baseline those — only property topics.

---

## Make resubscribe idempotent, and run it on the main loop

Two separate traps when a settings change switches scenario/port:

1. **Every save resubscribes.** If `resubscribe()` unconditionally
   unsubscribes and re-subscribes, a save that didn't change the scenario
   still drops the subscription for a moment and **misses any event in the
   gap**. Track the current value and no-op when it's unchanged:

   ```c
   if (handler && sub_id && strcmp(new_id, current_id) == 0)
       return true;   /* unchanged: keep the live subscription */
   ```

2. **Resubscribe from the wrong thread.** Web/FastCGI handlers run on their
   own thread(s); the AXEvent callback runs on the GLib main loop. Calling
   `ax_event_handler_unsubscribe` / `_free` / `_new` from the handler thread
   races the dispatch on the main loop — a classic double-free / use-after-
   free (two concurrent saves, or a save landing as an event dispatches).
   **Marshal the resubscribe onto the main loop** and only ever touch the
   handler there:

   ```c
   static gboolean resub_idle(gpointer data) {
       char *id = data;
       event_subscriber_resubscribe(id);   /* now on the main loop */
       g_free(id);
       return G_SOURCE_REMOVE;
   }
   /* from the HTTP handler thread: */
   g_idle_add(resub_idle, g_strdup(new_id));
   ```

   Combined with idempotence, this also collapses duplicate saves safely.

---

## Subscribing to a non-existent scenario/port succeeds silently

`ax_event_handler_subscribe` validates the topic *shape*, not that a live
source exists. Subscribing to AOA `Device1Scenario999`, or an input port the
device doesn't have, returns success — the callback just never fires. Guard
it: after subscribing, confirm the scenario/port exists (AOA
`getConfiguration`, or `root.IOPort` enumeration) and log a `LOG_WARNING` if
not, and/or run a periodic "no events in N minutes" health check. See
[aoa-api-patterns.md](./aoa-api-patterns.md).

---

## Checklist

- [ ] Topic namespaces and key names copied from `GetEventInstances`, not guessed
- [ ] Source-key filter uses the key the device actually declares (type too)
- [ ] Port/index values stored device-native (0-based for `IO/Port`); UI maps for display only
- [ ] `disabled` sentinel is not a valid index value (`"0"` is a real port)
- [ ] First delivery after (re)subscribe is treated as a baseline for property topics
- [ ] `resubscribe()` is a no-op when the target is unchanged
- [ ] All `ax_event_handler_*` calls happen on the main loop (marshal from other threads)
- [ ] Non-existent scenario/port is detected and logged, not silently dead

---

## References

- [AXEvent API](https://developer.axis.com/acap/api/native-sdk-api/#axevent) —
  `ax_event_handler_subscribe`, `ax_event_key_value_set_*`, source vs. data keys.
- [AXIS OS event list / `GetEventInstances`](https://developer.axis.com/vapix/network-video/event-and-action-services/) —
  how to read the device's own topic and key declarations.
- [GLib main loop](https://docs.gtk.org/glib/main-loop.html) — `g_idle_add` for
  marshalling onto the dispatch thread.
- Related: [aoa-api-patterns.md](./aoa-api-patterns.md),
  [acap-security-audit.md](../standards/acap-security-audit.md) (Audit 7, concurrency).
