# Contributing

Corrections and additions are welcome. This knowledge base is only useful if it stays accurate, so contributions are held to a few standards.

## Principles

- **Cite official sources.** Technical claims should link to an authoritative source — [developer.axis.com](https://developer.axis.com/), [help.axis.com](https://help.axis.com/), or the [AxisCommunications GitHub org](https://github.com/AxisCommunications). Put them in a `## References` section at the end of the doc. When this repo and the official docs disagree, the docs win.
- **Label what's empirical.** Plenty of useful knowledge here is observed device behavior with no official documentation (firmware quirks, build-tool side effects, undocumented endpoints). That's fine — but flag it explicitly as empirical/observed and note the device model and AXIS OS version it was seen on, so readers can judge it.
- **Verify against firmware.** Where practical, confirm a claim on a real device before adding it, and say which firmware you tested.
- **Keep the split clean.** *Standards* are normative (what to do); *Guides* are descriptive (how a thing behaves). New how-it-works material belongs in `guides/`.
- **No secrets or environment specifics.** No device IPs, credentials, customer/project names, internal hostnames, or local filesystem paths. Use placeholders (`<device-ip>`, `<password>`).

## How to propose a change

1. Open an issue describing the correction/addition (link the official source if you have one).
2. For a new guide: keep it focused on one topic, add a `## References` section, and add a row to the table in [README.md](./README.md).
3. Submit a PR. Small, well-sourced changes merge fastest.

## License

By contributing you agree your contributions are licensed under the repository's [MIT License](./LICENSE).
