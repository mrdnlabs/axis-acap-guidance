# ACAP Development Knowledge Base

Standards, reusable prompts, and hard-won learnings for developing [Axis ACAP](https://developer.axis.com/acap/) applications — with a focus on agentic (AI-assisted) development workflows.

This is a vendor-neutral knowledge base distilled from real ACAP projects. It is not affiliated with or endorsed by Axis Communications. Technical claims were verified against official Axis documentation, but the platform moves quickly — **always confirm against the docs and your target firmware**, especially anything tied to a specific SDK or AXIS OS version.

## Standards

Documents that govern how ACAPs are scoped, built, and validated. Use them in order.

| # | Document | Purpose |
|---|----------|---------|
| 1 | [acap-project-suitability.md](./standards/acap-project-suitability.md) | **Gate 0.** Decide whether the proposed project is appropriate to build as a custom ACAP. Do this first. |
| 2 | [acap-project-go-to-prompt.md](./standards/acap-project-go-to-prompt.md) | **Kickstart.** Reusable prompt to begin an agentic development session with the right workflow. |
| 3 | [acap-development-standards.md](./standards/acap-development-standards.md) | **Build.** Technical standards: project structure, SDK preferences, manifest conventions, security requirements. |
| 4 | [acap-security-audit.md](./standards/acap-security-audit.md) | **Validate.** Six audit prompts and checklists to review code for security, packaging, and publish readiness. |

## Guides

General-purpose reference material for ACAP development. Written for a wide audience — any ACAP developer should find these useful.

### Build and Configuration
| Guide | Contents |
|-------|----------|
| [acap-manifest-gotchas.md](./guides/acap-manifest-gotchas.md) | Manifest pitfalls: schema/SDK version coupling, `settingPage` spelling, parameter group capitalization, `embeddedSdkVersion` semantics, credential param types, restart behavior |
| [acap-reverse-proxy-guide.md](./guides/acap-reverse-proxy-guide.md) | Complete reverse proxy guide: Apache forwards the full URI unchanged, manifest config, multiple access levels, debugging checklist |
| [acap-wsl-build-pitfalls.md](./guides/acap-wsl-build-pitfalls.md) | WSL2 build issues: 777 file permissions silently break the proxy, Docker BuildKit stale caches, CRLF line endings, Dockerfile template |
| [enable-unsigned-apps.md](./guides/enable-unsigned-apps.md) | Enabling unsigned ACAP installation via VAPIX `config.cgi` |

### VAPIX and Device APIs
| Guide | Contents |
|-------|----------|
| [vapix-local-auth-from-acap.md](./guides/vapix-local-auth-from-acap.md) | D-Bus `GetCredentials` pattern for local VAPIX calls from an ACAP — the `127.0.0.12` virtual host, credential lifecycle, common pitfalls |
| [axis-http-authentication-policy.md](./guides/axis-http-authentication-policy.md) | Basic vs Digest auth: the `Network.HTTP.AuthenticationPolicy` setting, why forcing Digest can 401 over HTTPS, negotiate-don't-force, and the TLS trust boundary |
| [vapix-audio-device-control.md](./guides/vapix-audio-device-control.md) | Mute/unmute audio outputs via `audiodevicecontrol.cgi` — nested channel path gotcha, topology discovery pattern |
| [dynamic-overlay-api.md](./guides/dynamic-overlay-api.md) | Dynamic text overlay API: `dynamicoverlay.cgi`, slot mapping `#D1`–`#D16`, when to use dynamic text vs. axoverlay |
| [aoa-api-patterns.md](./guides/aoa-api-patterns.md) | AXIS Object Analytics API: scenario discovery, countable types, count retrieval, field naming by transport, polling vs. events |

### Device-Specific Hardware
| Guide | Contents |
|-------|----------|
| [axis-c6110-hardware-notes.md](./guides/axis-c6110-hardware-notes.md) | C6110-E `dspcontrol` daemon and speaker-guard behavior, PipeWire node names, audio forwarding to C1110-E, paging console actions API |

## Reference

- **[Axis ACAP Native SDK examples](https://github.com/AxisCommunications/acap-native-sdk-examples)** — the official example repository (referenced throughout these guides).
- **[Axis ACAP documentation](https://developer.axis.com/acap/)** and **[VAPIX library](https://developer.axis.com/vapix/)** — primary sources. When a guide here and the official docs disagree, the docs win.

## License

MIT — see [LICENSE](./LICENSE).
