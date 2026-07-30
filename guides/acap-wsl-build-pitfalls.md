# ACAP Build Pitfalls on Windows (WSL2 and Git Bash)

Issues specific to building ACAPs with Docker on Windows. §1–3 and §5 are about **WSL2** with source
files on the Windows filesystem (`/mnt/c/`). §4 is about **Git Bash / MSYS**, a different environment
with a different failure.

---

## 1. File Permissions (777) Break ACAP Features

Windows filesystems mounted via WSL2 report all files as `rwxrwxrwx` (777). When these files are `COPY`'d into a Docker build and packaged into an `.eap`, the ACAP installer on the device may silently skip configuration steps.

**Known impact:** Apache reverse proxy rules are not created. The app installs and runs, but the proxy path returns 404. No error is logged anywhere.

### Fix

Add a chmod step in the Dockerfile between COPY and acap-build:

```dockerfile
COPY ./app /opt/app/
WORKDIR /opt/app

RUN find . -type f -exec chmod 644 {} + && \
    find . -type d -exec chmod 755 {} +

RUN . /opt/axis/acapsdk/environment-setup* && acap-build ./
```

This is safe to include unconditionally — it's a no-op on native Linux.

---

## 2. Docker BuildKit Caches Stale Files

Docker BuildKit on WSL2 sometimes does not invalidate the `COPY ./app` layer when source files under `/mnt/c/` change. The build appears to succeed, but the `.eap` ships with the **previous** version of `manifest.json` / source files. Observed failure modes:
- A manifest field you just edited (e.g. changing `paramConfig` `type` from `String` to `password:maxlen=127`) is not reflected in the built `.eap`. Verify with `tar -xOzf myapp.eap manifest.json`.
- Old `.eap` files in the build context get picked up by `acap-build`, which merges their manifest fields into the new build.

### Fix

Pass `--no-cache` to `docker build`. A full ACAP rebuild is a few seconds — not worth the diagnostic cost of a poisoned cache. Bake it into your build script:

```bash
docker build --no-cache \
  --build-arg ARCH="${ARCH}" \
  -t "myapp:${ARCH}" .
```

Also add a `.dockerignore` to prevent old `.eap` artifacts from polluting the build context:

```dockerignore
*.eap
*.o
*.log
.env.devices
.git
docs
```

(`.dockerignore` with `*.eap` excludes EAPs in both `./` and `./app/`; the older `app/*.eap` pattern misses root-level builds.)

If `--no-cache` is still not enough on a particular WSL2 install (rare but documented), copy the project to `/tmp/` and build from the native filesystem:
```bash
cp -r /mnt/c/myproject /tmp/myproject
docker build --no-cache -t myapp /tmp/myproject
```

### Verification

After every build that touches `manifest.json`, sanity-check the packaged manifest matches the source:
```bash
diff <(jq -S . app/manifest.json) <(tar -xOzf app/myapp.eap manifest.json | jq -S .)
```
Empty diff = good. Any difference = stale cache; rebuild with `--no-cache`.

---

## 3. Line Endings (CRLF vs LF)

Windows editors may save files with CRLF line endings. Shell scripts (`build.sh`, postinstall scripts) will fail with `/bin/bash^M: bad interpreter` if they have CRLF endings.

### Fix

Configure git:
```bash
git config --global core.autocrlf input
```

Or fix individual files:
```bash
sed -i 's/\r$//' build.sh
```

---

## 4. Not WSL: Git Bash / MSYS rewrites container paths

If you build on Windows from **Git Bash** (MSYS2) rather than WSL2, a different problem appears.
MSYS path conversion rewrites anything that looks like a Unix path in a command argument into a
Windows path — including paths that were meant for the *container*, not the host.

```bash
docker run --rm --entrypoint /bin/bash <image> -c "ls /opt/axis"
```

fails with:

```
exec: "C:/Program Files/Git/usr/bin/bash": stat C:/Program Files/Git/usr/bin/bash:
no such file or directory
```

`/bin/bash` was rewritten to a host path before Docker ever saw it. The same applies to `-v`
destinations, `--entrypoint`, and any absolute path in the command you pass.

### Fix

Disable path conversion for the call:

```bash
MSYS_NO_PATHCONV=1 docker run --rm --entrypoint /bin/bash <image> -c "ls /opt/axis"
```

Export it once per script if you make several such calls, or run Docker from PowerShell instead,
where the problem does not exist.

This bites hardest when inspecting an SDK image — reading headers, listing `pkg-config` packages,
listing manifest schemas — which is exactly when you are already unsure whether the problem is your
code or your toolchain. See
[sdk-firmware-version-matching.md](./sdk-firmware-version-matching.md) for those inspection
one-liners.

---

## 5. Recommended Dockerfile Template for WSL

```dockerfile
ARG ARCH=armv7hf
ARG VERSION=12.9.0
ARG UBUNTU_VERSION=24.04

FROM axisecp/acap-native-sdk:${VERSION}-${ARCH}-ubuntu${UBUNTU_VERSION}

COPY ./app /opt/app/
WORKDIR /opt/app

# Fix WSL file permissions
RUN find . -type f -exec chmod 644 {} + && \
    find . -type d -exec chmod 755 {} +

RUN . /opt/axis/acapsdk/environment-setup* && acap-build ./
```

Pair with `.dockerignore` (use `*.eap`, not `app/*.eap`, so root-level builds are excluded too — see §2):
```dockerignore
*.eap
*.o
.env.devices
.git
docs
```

---

## References

The pitfalls in this guide are **empirical** — observed when building with Docker on Windows: §1–3 and §5 on WSL2 against the Windows filesystem (`/mnt/c/`), and §4 on Git Bash (MSYS2) while building for an AXIS Q3538-SLVE on AXIS OS 12.11.77. They are not Axis-documented behavior; the underlying tooling is, though:

- [Build, install and run an ACAP application](https://developer.axis.com/acap/develop/build-install-run/) — the official `acap-build` / Docker build flow these notes work around.
- [`axisecp/acap-native-sdk` on Docker Hub](https://hub.docker.com/r/axisecp/acap-native-sdk) — the SDK images referenced in the `FROM` line.
- WSL filesystem behavior is general (not Axis-specific): see Microsoft's [WSL file-permissions docs](https://learn.microsoft.com/en-us/windows/wsl/file-permissions) for why `/mnt/c/` files report `777`.
- MSYS path conversion is likewise general: see the [MSYS2 filesystem-paths documentation](https://www.msys2.org/docs/filesystem-paths/) for the rewriting rules and `MSYS_NO_PATHCONV`.
