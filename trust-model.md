# MemScope Trust Model

This page states what MemScope does and does not do with your data and
your network. Every claim here is verifiable against the installed
wheel — see [Auditing these claims](#auditing-these-claims) below.

Companion documents: [privacy.md](privacy.md) for the full privacy and
GDPR posture, and the `LICENSE` file shipped in the distribution for
the binding terms.

## Local-Only By Default

- Build artifact analysis is local-only.
- Remote/network artifact paths (URLs, UNC paths) are rejected for both
  artifacts and config files.
- No binary upload path exists in the analysis pipeline.

## Network Interaction Boundaries

- MemScope makes **no network calls of any kind** in normal operation.
- There is **no licensing backend, no activation, no licence check, no
  update check, and no usage reporting**. MemScope is free for any use
  including commercial use, so there is nothing to verify against a
  server — the code that would do so is not in the distribution.
- The **only** outbound path is opt-in telemetry, described below.

## Telemetry and Monitoring

- Telemetry is **disabled by default**. Omitting the `[telemetry]` block
  from `memscope.toml` means no request is ever issued.
- When you enable it, you must set **both** `enabled = true` and an
  `endpoint` you control. Setting `enabled = true` without an endpoint
  is a configuration error and MemScope refuses to load the config
  rather than guessing a destination.
- The payload is exactly three fields: `version`, `command`, and
  `runtime_ms`. A whitelist enforced in code drops anything else before
  transmission. No file paths, no target names, no toolchain identifier,
  no symbol or section names, no analysis results, no machine or install
  identifier.
- The request goes to **your** endpoint. The Licensor neither receives
  nor stores telemetry data.
- POSTs are best-effort with a 1-second timeout; failures are dropped
  silently and never affect the exit code.

## Privacy Guarantees

- Reports are generated locally as JSON/HTML/CSV files and are never
  transmitted.
- Reports carry no licence identifiers, no watermarks, and no
  attestation footer.
- Diagnostics bundle export sanitizes filesystem paths and redacts
  sensitive metadata fields (tokens, keys, fingerprints, signatures,
  install identifiers).
- MemScope writes two local files under `~/.memscope/`: a
  terms-acceptance record and a per-install UUID. Neither is
  transmitted. See [privacy.md](privacy.md) for the full table.

## Offline Operation

- Analysis works fully offline, permanently, with no degradation.
- There is no activation, no grace period, and no periodic revalidation
  — nothing expires and nothing needs to phone home. An air-gapped
  build agent runs MemScope indefinitely with no special configuration.

## Auditing These Claims

```bash
# Linux/macOS — observe syscall-level network activity:
strace -e trace=network -f memscope analyze ...

# Or run under a default-deny firewall and observe what gets blocked.
```

Without telemetry opted in you should see **no outbound connections at
all**. If you see any, that is a bug — please report it to
<dumitrescu.adrian121@gmail.com>.

You can also inspect the installed package directly: there is no
licensing or activation module in `site-packages/memscope/`. The
release pipeline asserts this on every published wheel
(`scripts/wheel_smoke_test.py`).
