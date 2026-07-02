# ACAP Lifecycle and Cleanup

Bringing an ACAP up in the right order is easy. Bringing it down cleanly, under a signal, from a mix of GLib main-loop code and detached worker threads, without racing D-Bus, without leaving a stateful event set, without leaking curl handles or use-after-freeing your token queue — that's where most first-cut ACAPs bleed.

This guide is the concrete pattern that survives all of those.

---

## What the failure modes actually look like

You almost never notice these on happy-path testing. The signals fire during:
- `systemctl restart myacap` from the operator UI
- `control.cgi?action=restart&package=myacap` from a config-changer script
- `axparhandd` restart during a firmware upgrade
- `SIGTERM` from the SDK harness when the ACAP crashes and respawns

The symptoms:
- **Systemd SIGKILL mid-cleanup.** Your `ax_event_handler_free()` or `ax_parameter_free()` blocks on a stuck D-Bus and the systemd unit timeout expires first. Half your cleanup didn't run; the next start finds stale state.
- **Use-after-free in a detached alarm/webhook thread.** The signal arrived while a curl `POST` was still in flight. Your cleanup ran `curl_global_cleanup()` and freed the token mutex and the AXParameter handle; the worker returned from `curl_easy_perform` and touched all three. Crash, respawn, sometimes core dump.
- **UAF in an AXEvent callback.** A CivetWeb worker thread called `event_subscriber_resubscribe()` (which frees + recreates the event handler) at the same moment the main loop dispatched the AXEvent callback. Old handler pointer freed, new one not yet installed, callback runs on the free.
- **Stuck stateful event.** The stateful "alarm active" event was set high and cleanup never explicitly set it low. Axis event rules see the alarm as still firing until the next reboot.

---

## The pattern that works

Four concrete rules. Together they close all four failure modes above.

### Rule 1: cleanup order mirrors init order, with two specific tweaks

Init order: config → token-manager → alarm/curl → event-publisher → web-server → main-loop → event-subscriber → input-trigger.

Cleanup order — **not** the exact reverse. Two things move:

- **Web server first.** Not last. `mg_stop()` closes all incoming HTTP connections and joins the worker threads BEFORE we start freeing modules that those handlers touch (AXEvent handlers via `resubscribe`, AXParameter via `config POST`). If web-server runs last, an in-flight config POST can call into an already-freed AXEvent handler.
- **Alarm-worker drain before its dependencies.** Detached worker threads read `config_get_string(...)`, call `alarm_record_update(...)`, and hold curl handles. Drain them BEFORE `config_cleanup`, `token_manager_cleanup`, and `curl_global_cleanup`. See rule 2.

```c
web_server_cleanup();          /* mg_stop; joins CivetWeb workers      */
alarm_handler_drain(3000);     /* bounded wait on detached workers     */
input_trigger_cleanup();       /* AXEvent handler free                 */
event_subscriber_cleanup();    /* AXEvent handler free                 */
event_publisher_cleanup();     /* send stateful=false; handler free    */
alarm_handler_cleanup();       /* curl_global_cleanup                  */
token_manager_cleanup();       /* GMutex clear                         */
config_cleanup();              /* ax_parameter_free                    */
```

### Rule 2: track detached workers with a counter + condvar

Convert detached threads to counted, not joinable. A counter+condvar is simpler than a slot pool of `pthread_t`s and does everything you need for a bounded drain.

```c
static int             g_workers_inflight = 0;
static pthread_mutex_t g_workers_mtx      = PTHREAD_MUTEX_INITIALIZER;
static pthread_cond_t  g_workers_cv       = PTHREAD_COND_INITIALIZER;

static void workers_add(void) {
    pthread_mutex_lock(&g_workers_mtx);
    g_workers_inflight++;
    pthread_mutex_unlock(&g_workers_mtx);
}

static void workers_remove(void) {
    pthread_mutex_lock(&g_workers_mtx);
    if (g_workers_inflight > 0) g_workers_inflight--;
    pthread_cond_broadcast(&g_workers_cv);
    pthread_mutex_unlock(&g_workers_mtx);
}

/* Call workers_add() BEFORE pthread_create.  If create fails, workers_remove()
 * on the error path.  On success, the worker's exit path calls workers_remove(). */

void my_module_drain(int timeout_ms) {
    struct timespec deadline;
    clock_gettime(CLOCK_REALTIME, &deadline);
    deadline.tv_sec  += timeout_ms / 1000;
    deadline.tv_nsec += (long)(timeout_ms % 1000) * 1000000L;
    if (deadline.tv_nsec >= 1000000000L) { deadline.tv_sec++; deadline.tv_nsec -= 1000000000L; }

    pthread_mutex_lock(&g_workers_mtx);
    while (g_workers_inflight > 0) {
        int rc = pthread_cond_timedwait(&g_workers_cv, &g_workers_mtx, &deadline);
        if (rc != 0) {
            syslog(LOG_WARNING,
                   "myacap: %d worker(s) still in flight after %d ms; "
                   "leaving them (cleanup may race)",
                   g_workers_inflight, timeout_ms);
            break;
        }
    }
    pthread_mutex_unlock(&g_workers_mtx);
}
```

Reserving the slot _before_ `pthread_create` matters. Otherwise a signal that arrives between reserve and the worker's first executed line lets drain return without waiting; the worker then runs against a torn-down world.

### Rule 3: SIGALRM watchdog around the whole cleanup chain

`ax_event_handler_free` and `ax_parameter_free` are synchronous D-Bus calls. Under normal operation they return in milliseconds. If `axeventd` or the parameter daemon is wedged, they can block for 10+ seconds. `systemd` will `SIGKILL` you before then, leaving whatever state the last cleanup step touched half-torn-down.

An `alarm(5)` + `SIGALRM` handler that just `_exit(0)`s is enough:

```c
static void on_cleanup_alarm(int sig) {
    (void)sig;
    static const char msg[] = "myacap: cleanup exceeded watchdog, forcing exit\n";
    ssize_t r = write(STDERR_FILENO, msg, sizeof(msg) - 1);
    (void)r;
    _exit(0);
}

/* In main(), right before the cleanup chain: */
signal(SIGALRM, on_cleanup_alarm);
alarm(5);
/* ... cleanup ... */
alarm(0);   /* cancel if we made it through */
```

Only async-signal-safe calls in the handler: `write`, `_exit`. No `syslog`, no `printf`, no `pthread_*`.

Systemd's default kill timeout is 30 seconds but Axis ACAPs run under a tighter harness; the 5-second cap matches the observed real-world limit and covers a couple of stuck D-Bus calls.

### Rule 4: emit `stateful=false` before you free the publisher

If your ACAP publishes a stateful event (fence violation, tamper detected, badge missing) and you shut down while the event is _active_, subscribers see the active state forever. Explicitly send `active=false` in `event_publisher_cleanup()` before `ax_event_handler_free`.

The reason to put this in `event_publisher_cleanup` and not in the signal handler: the signal handler must only be async-signal-safe, and `ax_event_new_from_declaration` + `ax_event_handler_send_event` are not.

---

## The resubscribe race, and how far the pattern above closes it

Web handlers on CivetWeb worker threads sometimes call `event_subscriber_resubscribe(new_scenario)` in response to a config change. That function frees the old `AXEventHandler *` and creates a new one. At the same instant, the main loop may be dispatching the old handler's callback. Boom.

Rule 1 (web-server-cleanup runs first) closes the _shutdown_ window: `mg_stop` joins the CivetWeb workers before we reach `event_subscriber_cleanup`. But in _normal_ operation, a config POST followed by a rapid AXEvent still has a small window.

Fully closing it needs marshaling — the resubscribe runs on the GLib main-loop thread instead of the CivetWeb worker. Two patterns:

- **Fire-and-forget with `g_main_context_invoke`.** Cheapest. The web handler returns "config saved, resubscribe queued" without knowing whether the resubscribe succeeded. Fine if the operator UI polls status shortly after.
- **Synchronous marshal with `g_main_context_invoke` + GCond wait.** Web handler blocks until the main loop runs the callback and signals a condvar. Preserves the current API (POST returns the resubscribe result). Costs one condvar per POST.

Both are legitimate; pick based on whether the caller needs the result.

---

## Fallback config file — atomic writes, correct permissions

If your ACAP has a fallback config path (AXParameter unavailable → write to `localdata/config.<something>`), the same lifecycle discipline applies at write time. A crash mid-`fopen("w")` leaves the file empty or torn. On a device that stores an alarm-action password there, that's a corrupted-password bootloop.

Use `g_file_set_contents_full` with `CONSISTENT | DURABLE`:

```c
GError *err = NULL;
bool ok = g_file_set_contents_full(
    PATH, buf, len,
    G_FILE_SET_CONTENTS_CONSISTENT | G_FILE_SET_CONTENTS_DURABLE,
    0600, &err);
```

- `CONSISTENT`: write to `.tmp`, `rename` atomically.
- `DURABLE`: `fsync` before rename, `fsync` the parent directory after.
- Mode `0600` applies on create. On an existing file GLib preserves the previous mode across the rename, so `chmod(PATH, 0600)` afterwards is worth it to relock a file left world-readable by an older build.

Wrap it in the same module mutex you use around `ax_parameter_get/set` — the two paths are mutually exclusive at init time and one mutex covers both.

---

## Signal-handling in the main-loop era

- `SIGTERM` / `SIGINT` → `g_unix_signal_add`. The GLib source runs on the main loop, so quitting the loop from it is safe (`g_main_loop_quit` is documented safe from any thread but the signal-source pattern removes the doubt).
- `SIGPIPE` → `signal(SIGPIPE, SIG_IGN)` in `main`. See [libcurl-in-acap-checklist.md](./libcurl-in-acap-checklist.md).
- `SIGALRM` → the cleanup watchdog above. Only used inside the cleanup chain; not registered otherwise.

Do not register `SIGTERM` with `signal()` directly. `g_unix_signal_add` integrates with the main loop; `signal()` is async-signal-safe-only and can't call the rest of your cleanup chain.

---

## Checklist

- [ ] `main` cleanup order runs `web_server_cleanup` FIRST
- [ ] `main` cleanup runs `<workers>_drain(bound)` BEFORE freeing what the workers touch
- [ ] Detached workers use `workers_add/remove` around the whole thread lifetime, with `workers_add` BEFORE `pthread_create`
- [ ] `alarm(5) + SIGALRM → _exit(0)` bracket the cleanup chain
- [ ] Handler uses only async-signal-safe calls (`write`, `_exit`)
- [ ] `event_publisher_cleanup` explicitly emits `active=false` for any stateful event
- [ ] `signal(SIGPIPE, SIG_IGN)` at top of `main`
- [ ] `SIGTERM` / `SIGINT` via `g_unix_signal_add`, not `signal()`
- [ ] Fallback config uses `g_file_set_contents_full(CONSISTENT|DURABLE)` + `chmod 0600`
- [ ] Every module that touches AXParameter or a shared list under multiple threads is serialized with a mutex

---

## References

- [GLib `g_unix_signal_add`](https://docs.gtk.org/glib/func.unix_signal_add.html) — main-loop-integrated signal source
- [GLib `g_file_set_contents_full`](https://docs.gtk.org/glib/func.file_set_contents_full.html) — atomic + durable writes
- [POSIX signal-handler safety](https://man7.org/linux/man-pages/man7/signal-safety.7.html) — the list of safe calls
- Empirical: anti-tailgating ACAP commits `6141d30` (Phase C) — worker drain, cleanup order, watchdog; `1a1f091` (Phase E) — atomic fallback-config writes; `d56c738` (Phase B) — SIGPIPE ignore
