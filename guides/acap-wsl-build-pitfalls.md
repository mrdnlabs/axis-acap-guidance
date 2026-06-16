# ACAP Build Pitfalls on WSL2

Issues specific to building ACAPs with Docker on WSL2 with source files on the Windows filesystem (`/mnt/c/`).

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

## 4. Recommended Dockerfile Template for WSL

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
