# ACAP Project Go-To Prompt

> **What:** A reusable prompt (and recommended additions) to kick off agentic ACAP development sessions. Establishes the research-first, test-on-device, build-end-to-end workflow that produces reliable ACAPs.
>
> **Who:** Anyone starting an agentic development session for an ACAP project — paste or reference this prompt at the beginning of the conversation.
>
> **When:** At the start of each new ACAP project or major development session. Before using this prompt, confirm the project has passed the suitability assessment.
>
> **Related documents:**
> - [acap-project-suitability.md](./acap-project-suitability.md) — Gate 0: confirm the project is suitable before starting (do this first)
> - [acap-development-standards.md](./acap-development-standards.md) — Technical standards to follow during development
> - [acap-security-audit.md](./acap-security-audit.md) — Security and quality audit to run before publishing
> - [../guides/](../guides/) — Hard-won platform knowledge: VAPIX patterns, manifest/build gotchas, and device-specific notes

---

## Reusable prompt

I'm building an Axis ACAP. Please approach this as a full ACAP engineering task, not just a coding task.

Start by doing the following:

1. Research the latest relevant Axis ACAP SDK and manifest requirements for the target version I specify.
2. Review relevant official Axis documentation, VAPIX APIs, and Axis SDK/D-Bus interfaces.
3. Review GitHub ACAP examples and other high-signal reference implementations that are relevant to this project.
4. Inspect the local repo/workspace before making assumptions.
5. Use any test devices I provide for real validation: log in, inspect configuration, deploy, test, iterate, and verify behavior on-camera.
6. Prefer a fully on-camera solution unless I explicitly ask for a host-side bridge or workaround.
7. Before writing code, outline a brief testing approach: how will you verify the core feature works on-device? What API endpoints, log entries, or observable behaviors will you check? Design the application so its behavior is programmatically verifiable, not just visually inspectable.
8. Build, deploy, test, and fix issues end to end rather than stopping at analysis.
9. Save important findings, commands, caveats, and platform-specific learnings as `.md` files in a local `learnings/` directory.

Working style requirements:

- Treat live device testing as part of the task, not optional.
- When researching technical details, prefer primary sources and official Axis documentation.
- If there are multiple plausible implementation paths, choose the most production-sound path and explain the tradeoff briefly.
- Audit for publish-readiness before finishing: secrets, placeholder metadata, obsolete test code, dead code paths, stale docs, and packaging issues.
- Keep credentials in a local untracked file such as `.env.devices` and make sure it is ignored by git.
- If a device-specific or AXIS OS-specific issue appears, document it clearly in a local `learnings/` directory.
- If a workaround is used temporarily, call it out explicitly and keep pushing toward the proper on-camera solution.

Deliverables expected by default:

- working ACAP source
- buildable package artifacts
- deploy/test verification on real device(s)
- updated README
- short notes in a local `learnings/` directory for key learnings

Device credentials for agentic access:

I may provide credentials for test devices so you can deploy, configure, and verify ACAPs directly. Handle them with care:

- Store credentials only in `.env.devices` (or similar), never in source code, commit history, or conversation logs.
- `.env.devices` must be in `.gitignore` before any credentials are written to it.
- Prefer temporary or development-only credentials created specifically for the agentic session. These should be rotated or removed after development/testing is complete.
- Do not reuse production credentials for agentic access. If the only available credentials are production ones, flag this so I can create temporary alternatives.
- When the session is over, remind me to rotate or remove the credentials that were used.

Test devices I may provide:

- IP / hostname
- username / password (preferably temporary/dev-only)
- model information
- AXIS OS version if known

Please use them actively for:

- deployment
- log inspection
- app/service configuration
- API validation
- iteration and regression checks

Before wrapping up, tell me:

- what is complete
- what was verified on-device
- any remaining risks or optional follow-up work

---

## Recommended additions

These are worth appending when relevant:

### 1. Target matrix

Include the exact target models and architectures if you know them:

- `aarch64`
- `armv7hf`
- single-channel vs multi-channel devices

This helps force architecture-aware packaging and multi-channel design decisions earlier.

### 2. Release expectations

If you usually want shippable results, add:

> Treat this as a release candidate unless I say otherwise. Remove test-only paths, placeholder metadata, and obsolete migration code before finishing.

### 3. UI expectations

If there is any UI/web component, add:

> Build an operator-facing UI, not just a developer test UI. Prefer clear labels, sane defaults, inline help, and a production-ready visual design that fits Axis devices.

### 4. Widget/dashboard expectation

If your ACAPs often expose status externally, add:

> If a standalone widget or dashboard view would be useful, include one by default and expose its URL clearly in the main UI.

### 5. Notes discipline

If you want durable knowledge capture every time, add:

> Save findings in a local `learnings/` directory as you go, not only at the end. Separate research notes, live device notes, and publish/release notes when helpful.

### 6. Publish audit

If you always want the last-mile check, add:

> Before finishing, run a final publish audit covering credentials, ignored files, manifest metadata, dead code, stale scripts, README accuracy, and package artifacts.

### 7. Closed-loop verification

If the agentic tool has browser control, device access, or other instrumentation capabilities, add:

> Close the loop on your own work. Don't just build and deploy — verify that the application actually works by interacting with it the way an operator would. Use the tools available to you:
>
> - **Browser automation:** Navigate to the device web UI, open the ACAP settings page, confirm the UI renders correctly, change settings, and verify they take effect.
> - **VAPIX/API verification:** Call the ACAP's API endpoints and the device's VAPIX APIs to confirm behavior programmatically (e.g., check that a parameter change via the UI is reflected in `param.cgi`, or that an overlay update appears in the video stream metadata).
> - **Log inspection:** Read syslog output after deployment and after exercising features to confirm there are no errors, warnings, or unexpected behavior.
> - **Instrumentation apps:** If the ACAP produces output that is difficult to verify programmatically (audio playback, visual overlays, motion-triggered behavior), consider building a small companion test utility or using device APIs to measure the effect. For example: use the audio API to confirm a speaker is active, use analytics APIs to confirm an event was generated, or use a second camera's APIs to verify a cross-device integration.
>
> The goal is to catch integration issues, UI bugs, and behavioral errors during development — not after handoff. If you can deploy it, you can test it.

---

## Short version

If you want a shorter reusable version:

> Treat this as a full Axis ACAP engineering task. Research the latest ACAP SDK, official Axis docs, relevant VAPIX/D-Bus interfaces, and GitHub ACAP examples. Inspect the repo first, then use any test devices I provide to deploy, test, debug, and verify behavior on-camera. Prefer a fully on-camera solution. Build and validate end to end, update the README, keep secrets in an ignored local file, save key learnings in a local `learnings/` directory, and do a final publish audit before wrapping up.
