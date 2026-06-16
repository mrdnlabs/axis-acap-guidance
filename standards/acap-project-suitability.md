# ACAP Project Suitability Assessment

> **What:** A decision framework for evaluating whether a proposed application is appropriate to build as a custom ACAP using agentic development tools. This is Gate 0 — the first check before any development begins.
>
> **Who:** Anyone proposing, scoping, or approving an ACAP project — whether the developer, a project lead, or an agentic tool being asked to start building.
>
> **When:** Before starting development. If any question below results in a STOP or CAUTION outcome, resolve it before proceeding to the go-to prompt or writing any code.
>
> **Related documents:**
> - [acap-project-go-to-prompt.md](./acap-project-go-to-prompt.md) — Use this to start development after passing the suitability assessment
> - [acap-development-standards.md](./acap-development-standards.md) — Technical standards to follow during development
> - [acap-security-audit.md](./acap-security-audit.md) — Security and quality audit to run before publishing
> - [../guides/](../guides/) — Hard-won platform knowledge: VAPIX patterns, manifest/build gotchas, and device-specific notes

---

## How to Use

Work through each question below in order. Each has three possible outcomes:

- **GO** — No concern; proceed.
- **CAUTION** — Proceed only with the stated mitigation or explicit acknowledgment of the risk.
- **STOP** — Do not build this as a custom ACAP under these standards. Consider an alternative approach.

If you reach the end with all GO or resolved CAUTION outcomes, the project is suitable.

---

## Question 1: Human Safety

**Could a malfunction, crash, or incorrect output of this application put anyone at physical risk?**

Examples: access control (door locks, gates), fire/smoke response automation, medical device integration, vehicle or machinery control, evacuation signaling.

| Answer | Outcome |
|--------|---------|
| No | GO |
| Possibly, but only indirectly and with other safeguards in place | CAUTION — Document the safeguards that exist independently of this ACAP. The ACAP must not be the sole safety mechanism. |
| Yes, or it could become a safety dependency | STOP — Safety-critical functionality belongs in certified systems or device firmware, not in a custom ACAP. |

---

## Question 2: Privacy

**Does this application process, store, transmit, or expose personally identifiable information (PII), facial recognition data, audio recordings of conversations, or video content sent off-device?**

Examples: facial recognition enrollment, license plate databases, audio eavesdropping, streaming video to third-party cloud services, tracking individuals across cameras.

| Answer | Outcome |
|--------|---------|
| No — it works with device-local data (counts, metadata, overlays, device settings) | GO |
| It processes analytics metadata that could indirectly identify individuals (e.g., occupancy counts by zone) | CAUTION — Ensure no PII is stored or transmitted. Document what data leaves the device and why. |
| Yes — it handles PII, biometric data, or sends identifiable video/audio off-device | STOP — This requires privacy impact assessment, legal review, and likely compliance work (GDPR, CCPA, etc.) beyond the scope of these standards. |

---

## Question 3: Mission Criticality

**If this ACAP stops working (crash, bug, failed update, device reboot), what is the worst realistic consequence?**

Examples of low impact: a counter overlay goes blank, a status dashboard stops updating, an audio alert doesn't fire for a non-critical notification.

Examples of high impact: sole intrusion detection path goes offline, evidence recording stops, emergency notification system fails silently.

| Answer | Outcome |
|--------|---------|
| Loss of a convenience or efficiency feature; the core security/business function continues without it | GO |
| Noticeable operational gap, but redundant systems or manual processes cover it | CAUTION — Document the redundancy. The ACAP must not be marketed or relied upon as the primary path. |
| Significant operational, financial, or evidentiary harm; no redundant path exists | STOP — This needs a supported, warranted solution — not a custom utility ACAP. |

---

## Question 4: Regulatory and Compliance

**Is the use case subject to specific regulatory requirements (HIPAA, GDPR data processing, law enforcement evidence handling, financial audit, export control)?**

| Answer | Outcome |
|--------|---------|
| No — it's an internal operational utility | GO |
| Possibly — it touches data or systems adjacent to regulated processes | CAUTION — Get clarity on whether the ACAP itself falls within regulatory scope. If it does, STOP. |
| Yes — the application or its output is part of a regulated workflow | STOP — Regulated applications need formal validation, audit trails, and compliance certification that these standards do not provide. |

---

## Question 5: Scope

**Can you describe what the application does in one sentence?**

Try to complete this sentence: *"This ACAP ___."*

Good examples:
- "This ACAP displays AOA counter data as a large video overlay."
- "This ACAP plays a NOAA weather alert through the device speaker."
- "This ACAP forwards analytics events to an MQTT broker."

Warning signs:
- The sentence uses "and" more than once
- It requires a paragraph to explain
- It describes a "platform," "framework," "engine," or "system"
- It manages multiple devices, users, or tenants

| Answer | Outcome |
|--------|---------|
| One clear sentence; solves one specific problem | GO |
| One sentence, but it's doing two related things (e.g., "counts objects and sends alerts") | CAUTION — Consider whether it should be two separate ACAPs. If not, document why they must be combined. |
| Cannot be stated in one sentence; involves multi-device orchestration, user management, or platform-level functionality | STOP — This is beyond the scope of a utility ACAP. Consider a dedicated application, VMS integration, or server-side solution. |

---

## Question 6: Axis Technology Fit

**Can this be accomplished primarily using Axis SDK APIs, VAPIX, and on-device capabilities?**

| Answer | Outcome |
|--------|---------|
| Yes — the SDK provides the needed APIs (VDO, AXEvent, AXOverlay, VAPIX, etc.) | GO |
| Mostly — needs one or two external libraries (e.g., curl for HTTP, a JSON parser) alongside the SDK | GO — Use proven, well-maintained libraries. |
| Partially — requires significant custom infrastructure (custom ML runtime, custom protocol stack, custom database) | CAUTION — Each custom component is a maintenance and security burden. Justify why the Axis SDK alternative is insufficient. |
| Not really — the device is mostly a deployment target, and the real logic lives elsewhere | STOP — This is a server-side application deployed to a camera, not an ACAP. Consider running it on a proper server. |

---

## Summary

After working through all six questions, record the outcome:

| Question | Outcome | Notes |
|----------|---------|-------|
| 1. Human Safety | | |
| 2. Privacy | | |
| 3. Mission Criticality | | |
| 4. Regulatory/Compliance | | |
| 5. Scope | | |
| 6. Axis Technology Fit | | |

**Overall assessment:**

- All GO: Proceed to development using the [go-to prompt](./acap-project-go-to-prompt.md).
- One or more CAUTION (resolved): Proceed, with mitigations documented.
- Any STOP: Do not proceed under these standards. Revisit the approach.

---

## For Agentic Tools

If you are an AI assistant being asked to develop an ACAP, and the proposed project triggers a STOP on any question above, you should:

1. Clearly state which question(s) triggered the STOP and why
2. Suggest alternative approaches (e.g., server-side application, VMS integration, formal development process)
3. Do not proceed with development unless the user explicitly acknowledges the risk and overrides the assessment

For CAUTION outcomes, state the concern and the required mitigation, then proceed if the user confirms.
