# AXIS Object Analytics (AOA) API Patterns

How to interact with AXIS Object Analytics from an ACAP — scenario discovery, count retrieval, and common pitfalls.

---

## Overview

AXIS Object Analytics (AOA) runs as a separate application on the camera. An ACAP can query AOA for analytics data (counts, occupancy) via the local VAPIX CGI interface at:

```
/local/objectanalytics/control.cgi
```

Authentication is required. From an ACAP, use D-Bus `GetCredentials` with loopback (see [vapix-local-auth-from-acap.md](./vapix-local-auth-from-acap.md)).

The full request envelope documented by Axis is `{apiVersion, context, method, params}` — the `context` field is an optional caller-supplied echo value. The examples below use `apiVersion` `"1.2"` (the version on the current [AOA API docs](https://developer.axis.com/vapix/applications/axis-object-analytics-api/)); call `getSupportedVersions` to confirm what your device accepts.

---

## Countable Scenario Types

Not all AOA scenarios produce count data. The following scenario types support accumulated counts:

| Scenario Type | Count Method | Description |
|---------------|-------------|-------------|
| `crosslinecounting` | `getAccumulatedCounts` | Objects crossing a defined line |
| `occupancyInArea` | `getOccupancy` | Objects present in a defined area |

Other scenario types (e.g., `motion`) do not produce count data and should be filtered out when discovering countable scenarios.

---

## Scenario Discovery

Use `getConfiguration` to list all configured AOA scenarios, then filter to countable types:

```
POST /local/objectanalytics/control.cgi
Content-Type: application/json

{
    "method": "getConfiguration",
    "apiVersion": "1.2"
}
```

The response includes scenario objects with a `type` field. Select only scenarios where `type` is `crosslinecounting` or `occupancyInArea`.

---

## Retrieving Counts

### Cross-line counting

```json
{
    "method": "getAccumulatedCounts",
    "apiVersion": "1.2",
    "params": {
        "scenario": <scenario_id>
    }
}
```

### Occupancy in area

```json
{
    "method": "getOccupancy",
    "apiVersion": "1.2",
    "params": {
        "scenario": <scenario_id>
    }
}
```

---

## Available Count Fields

The response includes a `total` plus per-class fields. **Field naming is transport-dependent** — this trips people up:

| Transport | Per-class field names |
|-----------|----------------------|
| `control.cgi` method response (`getAccumulatedCounts` / `getOccupancy`) | **short** names: `human`, `car`, `bus`, `bike`, `truck`, `otherVehicle` (plus `total`) |
| Event / MQTT metadata payload (Crossline counting) | **prefixed** names: `totalHuman`, `totalCar`, `totalBus`, `totalBike`, `totalTruck`, `totalOtherVehicle` (plus `total`) |

So if you poll `control.cgi`, read `car`/`human`/etc.; if you subscribe to events/MQTT, read `totalCar`/`totalHuman`/etc. Verify the exact keys against your device's actual response.

**Note:** In both schemes there is no built-in `totalVehicle` (or `vehicle`) aggregate. To get a total vehicle count, sum the vehicle subclasses (e.g. `car + bus + bike + truck + otherVehicle` on `control.cgi`).

---

## Other Useful Methods

| Method | Purpose |
|--------|---------|
| `resetAccumulatedCounts` | Reset counters to zero (useful for testing) |
| `sendAlarmEvent` | Trigger a test alarm event (useful for validation) |

---

## Gotchas

- **Auto-selecting scenarios is risky.** If the ACAP automatically picks the first scenario, it may select a `motion` scenario that has no count data. Always filter to countable types first.
- **Scenario IDs are not stable across factory resets.** A factory default removes all AOA scenarios. The default configuration after reset typically includes only a `motion` scenario — any `crosslinecounting` or `occupancyInArea` scenarios must be recreated.
- **Authentication failures on loopback.** The documented virtual host for service-account credentials is `127.0.0.12` (see [vapix-local-auth-from-acap.md](./vapix-local-auth-from-acap.md)) — try that first. Note the AOA endpoint is served by the AOA ACAP rather than core VAPIX, so auth behavior here has been observed to differ by device model and firmware; if `127.0.0.12` returns `401`/`403`, probing `127.0.0.1` and `127.0.1.1` has empirically helped *on this endpoint specifically*. This is a workaround, not a documented pattern — don't generalize it to ordinary `axis-cgi` calls.
- **AOA must be running.** The AOA application must be installed and started on the device. If it's not running, the `/local/objectanalytics/` endpoint will not exist.
- **Polling vs. events.** Polling `getAccumulatedCounts` is simpler and more reliable for display purposes. The official Axis `modbus-acap` example uses event subscription instead, which avoids polling but requires more complex D-Bus/event integration.

---

## References

- [AXIS Object Analytics API (VAPIX)](https://developer.axis.com/vapix/applications/axis-object-analytics-api/) — the `/local/objectanalytics/control.cgi` endpoint; `{apiVersion, context, method, params}` request shape; `getConfiguration`, `getAccumulatedCounts`, `getOccupancy`, `resetAccumulatedCounts`, `sendAlarmEvent` methods; the distinct scenario `type` values `motion`, `fence`, `crosslinecounting`, `occupancyInArea`; and the short count-field names (`human`/`car`/`bus`/`bike`/`truck`/`otherVehicle` + `total`, no `totalVehicle`).
- [AXIS Object Analytics — counting data](https://developer.axis.com/analytics/axis-object-analytics/how-to-guides/axis-object-analytics-counting-data/) — the total-prefixed per-class field names (`totalHuman`/`totalCar`/…) used in event / MQTT metadata.
- [`modbus-acap` example](https://github.com/AxisCommunications/modbus-acap) — official example that consumes AOA via event subscription rather than polling.

> The loopback-address fallback note above (`127.0.0.1` / `127.0.1.1`) is an empirical workaround for this endpoint, not documented behavior — see [vapix-local-auth-from-acap.md](./vapix-local-auth-from-acap.md).
