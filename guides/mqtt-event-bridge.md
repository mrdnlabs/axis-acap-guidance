# MQTT event bridge: topics, payloads, and not clobbering the operator

The device's built-in MQTT client can publish device events — including events an ACAP declares
itself — to an external broker. That makes it the cheapest possible integration path for a VMS: the
application declares AXEvents and holds no broker credentials, TLS material or connection state at
all.

Two things are worth knowing before you rely on it: the **exact** topic and payload shape it
produces, and the fact that the configuration call **replaces the entire filter list**.

> **Empirical unless cited.** Topic and payload shapes below were captured from a live Mosquitto
> broker fed by an AXIS Q3538-SLVE on AXIS OS 12.11.77, MQTT API version 1.6.

---

## 1. Topic format

```
<deviceTopicPrefix>/event/<namespaced event topic>[/$source/<key>/<value>]
```

Real captures:

```
axis/<serial>/event/tns:axis/CameraApplicationPlatform/ObjectAnalytics/Device1Scenario1
axis/<serial>/event/tns:onvif/Device/tns:axis/IO/Port/$source/port/1
axis/<serial>/event/tns:axis/CameraApplicationPlatform/MyApp/Exited/$source/zone/3
```

Details that matter:

- The default `deviceTopicPrefix` is **`axis/<serial>`**, and the bridge appends **`/event`**. Read
  the actual value from `getClientStatus` rather than assuming.
- With `includeTopicNamespaces: true` (the default), namespaces render with a **colon** —
  `tns:axis`, not `tnsaxis`. This is a common transcription error when hand-writing a subscription.
- **ONVIF source keys are rendered into the topic path** as `$source/<key>/<value>`.

That last point is the useful one. If your event declares a source key, subscribers get a
**per-instance topic** for free — a VMS can subscribe to one zone, one port, one channel, rather
than filtering payloads:

```
.../MyApp/Exited/$source/zone/3        # just zone 3
.../MyApp/Exited/#                     # every zone
```

If you are designing an ACAP's events and expect consumers to care about which instance fired,
declare that discriminator as an AXEvent **source** key (`ax_event_key_value_set_mark_as_source()`)
rather than a data key.

## 2. Payload format

```json
{
  "topic": "onvif:Device/axis:IO/Port",
  "timestamp": 1785325150144,
  "message": {
    "source": { "port": "1" },
    "key": {},
    "data": { "state": "0" }
  }
}
```

Two consequences that will bite an integration contract:

- **Every value is a string.** A `double` declared as `AX_VALUE_TYPE_DOUBLE` arrives as
  `"112.000000"`; a `bool` arrives as `"1"` or `"0"`. Do not promise consumers JSON numbers or
  booleans, and do not expect a strict schema validator on the far end to accept them.
- **`timestamp` is epoch milliseconds**, not ISO-8601. If a consumer needs a human-readable or
  sub-millisecond UTC time, emit your own timestamp field in the event data — the bridge's own value
  is not that.

Source keys appear both in the topic path *and* in `message.source`.

## 3. `configureEventPublication` replaces the whole filter list

The publication config is a single list. `configureEventPublication` sets it wholesale — it does not
merge. **Posting only your own filters silently deletes every filter the operator configured.**

For an ACAP that configures the bridge on the user's behalf, the write must be read-merge-write:

1. `getEventPublicationConfig` — read the current list.
2. Drop only entries you previously added (match on something identifying, e.g. your own topic
   prefix), so repeated runs stay idempotent instead of accumulating duplicates.
3. Append your filters.
4. Post the merged list back, preserving everything else untouched.

```bash
# read
curl --anyauth -u "root:<password>" -X POST -H "Content-Type: application/json" \
  -d '{"apiVersion":"1.0","method":"getEventPublicationConfig"}' \
  "http://127.0.0.12/axis-cgi/mqtt/event.cgi"
```

A clean device returns an **empty** `eventFilterList`, which is a useful restore target:

```json
{"topicPrefix":"default","customTopicPrefix":"","appendEventTopic":true,
 "includeTopicNamespaces":true,"includeSerialNumberInPayload":false,"eventFilterList":[]}
```

One subtree filter covers every event under a topic prefix, which makes add/remove trivially
idempotent:

```json
{"topicFilter":"axis:CameraApplicationPlatform/MyApp//.","qos":0,"retain":"none"}
```

### Global flags are device-wide — read them, don't write them

`topicPrefix`, `customTopicPrefix`, `appendEventTopic` and `includeTopicNamespaces` apply to
**every** event the device publishes, not just yours. An application that "helpfully" sets a custom
prefix silently rewrites the topics of every other integration on that camera.

Read them and adapt what you display to the operator. Only write them on an explicit, informed
opt-in.

## 4. Verify the topic string against a real broker

The topic your application *predicts* and the topic a broker *receives* must be diffed, not assumed —
namespace rendering and source-key placement are exactly the sort of thing to get subtly wrong, and a
wrong string copied into a VMS is expensive to debug from the far end.

```bash
# subscribe to everything the device publishes
mosquitto_sub -h <broker> -t '#' -v
```

Then fire the event and compare byte-for-byte with whatever your UI or docs tell integrators to use.
If your ACAP has a test-event trigger, this takes seconds and needs nothing in the camera's view.

For a lab broker, anonymous access on plain 1883 is fine and keeps the test cheap — but it is a lab
fixture, not a production configuration:

```conf
listener 1883 0.0.0.0
allow_anonymous true
log_type all
```

## 5. Prerequisites and failure modes

- The operator must have **configured and enabled the device MQTT client** (`System > MQTT`, or
  `mqtt/client.cgi`). Publication config alone publishes nothing. Check
  `getClientStatus` → `status.state == "active"` and `status.connectionStatus == "connected"` and say
  which one is missing, rather than reporting a generic failure.
- Configuring the bridge from inside an ACAP is a local VAPIX call, so it needs D-Bus
  service-account credentials against the `127.0.0.12` virtual host — see
  [vapix-local-auth-from-acap.md](./vapix-local-auth-from-acap.md). Declare the method in the
  manifest:

  ```json
  "resources": {
      "dbus": { "requiredMethods": ["com.axis.HTTPConf1.VAPIXServiceAccounts1.GetCredentials"] }
  }
  ```
- Events declared by an ACAP are ordinary device events, so they are publishable the same way — no
  separate mechanism, and no need for the application to embed an MQTT client of its own.

---

## References

- [VAPIX — MQTT event bridge](https://developer.axis.com/vapix/network-video/mqtt-event-bridge/) —
  `mqtt/event.cgi`, `configureEventPublication`, `getEventPublicationConfig`, `topicPrefix`,
  `customTopicPrefix`, `appendEventTopic`, `includeTopicNamespaces`, per-filter `qos` and `retain`.
- [VAPIX — MQTT client API](https://developer.axis.com/vapix/network-video/mqtt-client-api/) —
  `mqtt/client.cgi`, `getClientStatus`, `deviceTopicPrefix`, `activateClient`.
- [Event API (AXEvent)](https://developer.axis.com/acap/api/native-sdk-api/#event-api) —
  declaring events, and `ax_event_key_value_set_mark_as_source()` for the source keys that become
  topic path segments.
- [Device integration with MQTT (white paper)](https://whitepapers.axis.com/en-us/device-integration-with-mqtt) —
  background on the device MQTT client.
- Topic and payload *shapes* in this guide are **observed** captures, not quoted from the reference.
