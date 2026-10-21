# Match the SDK to your target firmware — and read the headers

Two related traps: picking an SDK version that does not correspond to the firmware you are deploying
to, and copying an official example that targets a newer SDK than yours. Both produce confusing
failures — sometimes a compile error naming symbols that exist in the documentation, sometimes a
package that installs and then does not work.

> **Empirical unless cited.** The API drift table was produced by diffing SDK headers between
> `axisecp/acap-native-sdk:12.10.0-aarch64-ubuntu24.04` and `:12.11.0-aarch64-ubuntu24.04`, while
> targeting an AXIS Q3538-SLVE on AXIS OS 12.11.77.

---

## 1. From 12.x, the SDK version tracks the AXIS OS version

| ACAP Native SDK | AXIS OS |
|---|---|
| 12.11 | 12.11 |
| 12.10 | 12.10 |
| 12.0 | 12.0 |
| 1.15 | 11.11 |
| 1.13 | 11.9 |

Before 12.0 the numbering was offset; from 12.0 it is 1:1. So the SDK tag in your `FROM` line is
effectively a statement about which AXIS OS you are building for:

```dockerfile
FROM axisecp/acap-native-sdk:12.11.0-aarch64-ubuntu24.04
```

**Each SDK also has its own minimum AXIS OS**, which is more specific than the major.minor. SDK
12.11.0 reports a minimum of **12.11.72** — setting `compatibleOsVersions.min` below that produces a
build warning:

```
* Warning: compatibleOsVersions entry 0: 'min' is set to '12.11', which is below
  the SDK's minimum AXIS OS version '12.11.72'
```

Take the warning literally rather than rounding it off; from AXIS OS 12.10 `compatibleOsVersions`
also gates installation (see [acap-manifest-gotchas.md](./acap-manifest-gotchas.md)).

## 2. The official examples target the newest SDK

[acap-native-sdk-examples](https://github.com/AxisCommunications/acap-native-sdk-examples) tracks the
current release. Its Dockerfiles pin a version:

```dockerfile
ARG VERSION=12.11.0
```

If your fleet is a release behind, an example copied verbatim **will not compile**, and the errors
name functions that exist perfectly well in the online documentation — because the documentation
describes the newer SDK. Check the `ARG VERSION` in the example's Dockerfile before assuming the code
matches your target.

## 3. Case study: the Device Data Hub C API changed between 12.10 and 12.11

This is not a case of new functions being added. Existing names, types and call shapes changed:

| Concern | SDK 12.10 | SDK 12.11 |
|---|---|---|
| Error type | `DHClientError`, `dh_client_error_to_string()`, `dh_client_destroy_error()` | `DHError`, `dh_error_to_string()`, `dh_error_destroy()` |
| Callbacks | build a listener object: `dh_subscriber_listener_create()` + `dh_subscriber_set_listener()` | set directly: `dh_subscriber_set_data_callback()`, `dh_subscriber_set_topic_update_callback()` |
| Filtering | none — topic names passed straight to `dh_subscriber_subscribe()` | `DHFilter` + `DHSubscribeOptions` objects |
| Sample accessors | `dh_topic_sample_get_topic_data()`, `dh_topic_data_get_json_str()` | `dh_topic_sample_get_data()`, `dh_topic_data_get_json_data()` |
| Data callback argument | `DHTopicSample*` | `const DHTopicSample*` |
| Topic list release | `dh_client_destroy_topic_list()` | `dh_topic_list_destroy()` |
| `dh_client_create` | `(name, options)` — no error out-parameter | `(name, DHError**)` |
| `dh_client_is_connected` | present | **absent** — use `dh_client_get_connection_state()` |

The manifest resource key moved too: `deviceDataHub_beta2` in manifest schema 2.0.0 (the newest
schema SDK 12.10 ships) versus `deviceDataHub` in 2.2.0.

Two lessons generalize beyond this API:

- An SDK API still marked beta can change shape between adjacent minor releases.
- Isolate the SDK surface behind one module. Porting the case above was contained to a single
  file precisely because nothing else called those functions directly.

## 4. Read the headers in the image you are building with

The authoritative answer for *your* build is in the sysroot, not online. It costs seconds:

```bash
# what does this SDK actually declare?
docker run --rm --entrypoint /bin/bash \
  axisecp/acap-native-sdk:12.11.0-aarch64-ubuntu24.04 \
  -c "cat /opt/axis/acapsdk/sysroots/aarch64*/usr/include/datahub/subscriber.h"
```

Useful variations:

```bash
# which libraries are available to pkg-config?
docker run --rm --entrypoint /bin/bash axisecp/acap-native-sdk:12.11.0-aarch64-ubuntu24.04 \
  -c "ls /opt/axis/acapsdk/sysroots/aarch64*/usr/lib/pkgconfig/ | grep -iE 'ax|mdb|datahub|vdo'"

# which manifest schemas does this SDK ship?
docker run --rm --entrypoint /bin/bash axisecp/acap-native-sdk:12.11.0-aarch64-ubuntu24.04 \
  -c "ls /opt/axis/acapsdk/axis-acap-manifest-tools/schema/schemas/*/"
```

> On Windows under Git Bash, container paths in `docker run` get rewritten by MSYS path conversion —
> prefix with `MSYS_NO_PATHCONV=1` or use PowerShell. See
> [acap-wsl-build-pitfalls.md](./acap-wsl-build-pitfalls.md).

## 5. Third-party SDK headers may not survive your warning flags

The Axis example Makefiles use a strict set including `-Wpedantic -Werror`. Some SDK headers do not
satisfy it — the VDO headers use enumerator values outside the range of `int`:

```
/opt/axis/acapsdk/sysroots/aarch64/usr/include/vdo/vdo-types.h:222:23: error:
  ISO C restricts enumerator values to range of 'int' before C2X [-Werror=pedantic]
```

Relax the flag for the one translation unit that includes the offending header rather than dropping
it across your own code:

```make
# The VDO SDK headers use enumerator values outside the range of int, which
# -Wpedantic rejects. Relax it only where they are included.
overlay.o: CFLAGS += -Wno-pedantic

%.o: %.c
	$(CC) -c $< $(CFLAGS) -o $@
```

---

## References

- [Build, install and run an ACAP application](https://developer.axis.com/acap/develop/build-install-run/) —
  the Docker/`acap-build` flow and SDK image tags.
- [`axisecp/acap-native-sdk` on Docker Hub](https://hub.docker.com/r/axisecp/acap-native-sdk) —
  available SDK tags; the tag is how you select the target AXIS OS.
- [ACAP Native SDK API list](https://developer.axis.com/acap/api/native-sdk-api/) — includes the note
  that the Message Broker API is removed in AXIS OS 13.0 and that the Device Data Hub API arrived in
  SDK 12.10.
- [acap-native-sdk-examples](https://github.com/AxisCommunications/acap-native-sdk-examples) — check
  each example's `ARG VERSION` against your target firmware.
- [Manifest schema version history](https://developer.axis.com/acap/reference/manifest-schemas/) —
  which schema versions exist and when they were introduced.
