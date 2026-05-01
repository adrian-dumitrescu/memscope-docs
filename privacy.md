# MemScope privacy and telemetry policy

> **Effective:** 2026-04-30 — applies to MemScope (`memscope-fw` on PyPI)
> v0.1.0a17 and later.
> **Data controller:** Adrian Dumitrescu (Romania), to be assigned to
> the Licensor's commercial entity upon registration.
> **Contact for privacy / data-subject requests:** dumitrescu.adrian121@gmail.com

MemScope is a **local-first** tool: it runs against your firmware artifacts
on your machine, writes reports to your local filesystem, and **does not
phone home or transmit any data by default**.

The free `memscope-fw` core distribution on PyPI makes **no network
calls of any kind** in normal operation. There is no licensing backend
contact, no telemetry, no update check, no usage reporting. Anyone can
install, run `memscope analyze`, and never have a single byte leave
their machine.

The only situations where MemScope contacts the network are:

1. **Subscription Bundle activation / refresh** — only relevant if you
   have purchased an optional commercial Bundle from the Licensor
   (delivered as a Customer-Bound Wheel separately from PyPI). The
   subscription Bundle's activation flow contacts the licensing
   backend (Keygen.sh) on first install and on a periodic refresh
   interval. **Bundle licences are not yet generally available for
   sale**, so this path is dormant for almost all users today.
2. **Opt-in telemetry**, described below, which is **OFF by default**
   and requires explicit configuration to enable.

## GDPR posture

For users in the European Economic Area, the United Kingdom, and other
jurisdictions with comparable data-protection regimes:

- **Free core (everyone):** MemScope makes no network calls and
  processes no personal data on the Licensor's behalf. The Licensor is
  neither a data controller nor a processor with respect to free-core
  use.
- **Subscription Bundles only:** the Licensor (via the Keygen.sh
  sub-processor) processes the **machine fingerprint hash**,
  **install ID**, **OS summary**, and **License Key** described in
  [What the licensing backend sees](#what-the-licensing-backend-sees).
  Under GDPR Article 6(1)(b) this processing is necessary for the
  performance of the License Agreement (clause 4 of the LICENSE).
  The Licensor acts as **data controller** for this processing;
  Keygen.sh acts as the Licensor's data **processor** under a Data
  Processing Agreement (DPA) executed between the Licensor and
  Keygen, Inc. **One-Time Bundles and Bespoke Bundles make no network
  calls** — they are identical to the free core for privacy purposes.
- **Opt-in telemetry:** when you enable telemetry by setting
  `[telemetry].endpoint` in your `memscope.toml`, the **endpoint you
  configure** is the recipient. The Licensor neither receives nor
  stores telemetry data — it only ships the code that POSTs to the
  endpoint of your choosing. You are the data controller for any
  data sent to your own collector.
- **Data-subject rights:** you may request access, rectification,
  erasure, restriction, portability, or object to processing of
  personal data the Licensor controls (the Subscription Bundle
  activation/refresh payload). Contact: dumitrescu.adrian121@gmail.com.
  The Licensor responds within 30 days.
- **Sub-processor:** Keygen, Inc. (US, with EU data residency
  available). See the Sub-processors section below.
- **Data Processing Addendum (DPA):** enterprise customers entering
  into Bundle contracts may request a DPA in writing. A draft template
  is referenced in the LICENSE preamble.

## Sub-processors (Subscription Bundles only)

| Sub-processor | Role | Data shared | Region |
|---|---|---|---|
| Keygen, Inc. (https://keygen.sh) | Subscription Bundle activation, validation, revocation | License Key, machine fingerprint hash, install ID, OS summary, Bundle name + version | US (default) — EU residency available on request |

No other sub-processors are used by Subscription Bundles. The free
core, One-Time Bundles, and Bespoke Bundles use no sub-processors at all.

## What the licensing backend sees

(Subscription Bundles only.) When you activate a Subscription Bundle
licence key, MemScope sends to Keygen:

- The License Key you typed.
- A **machine fingerprint hash** (SHA-256 of your hostname + OS + machine
  arch + a per-install UUID). This is a stable identifier for your
  machine but it is not reverse-engineerable to identify you personally.
- An OS summary: `system`, `release`, `version`, `machine`, `python_version`.
  Same data `python -c "import platform; print(platform.uname())"` reveals.
- An install ID (UUID generated on first run, stored at `~/.memscope/install_id`).
- The Bundle name and version.

Keygen does NOT see: file paths, symbol names, target names, toolchain
identifiers, source code, or report contents.

You can audit the exact payload at runtime by capturing network traffic
during activation (see [Auditing what we just said](#auditing-what-we-just-said) below).

## Opt-in telemetry

If you enable telemetry, MemScope reports a tiny payload to an HTTPS
endpoint of YOUR choosing (you control where the data goes):

### What is collected (and only this)

| Field | Example | What it tells us |
|---|---|---|
| `version` | `0.1.0a0` | Which release of MemScope is in use |
| `license_mode` | `active` | trial / active / offline-valid / etc — coarse mode only |
| `license_tier` | `paid` | trial / paid / developer / unavailable |
| `command` | `analyze` | Which subcommand ran (analyze, diff, validate) |
| `runtime_ms` | `4521` | How long the command took, in milliseconds |
| `exit_code` | `0` | Whether the command succeeded |

### What is NEVER collected

- File paths (ELF, MAP, linker, output paths).
- Target name (project / firmware identifier).
- Toolchain identifier.
- Memory region names, section names, symbol names.
- Issue counts, byte counts, or any analysis output.
- Hostname, IP address, MAC address, or other network identifiers.
- License key (the full key never leaves your machine after activation).
- Machine fingerprint (the hash is sent only to the license backend, not
  to telemetry).
- Personally identifying information of any kind.

The whitelist of allowed fields is enforced in code. Any future caller
that passes a non-whitelisted field has it silently dropped before
transmission. You can verify by running with telemetry enabled against
a request-logging endpoint of your own.

### How to enable

In your `memscope.toml`:

```toml
[telemetry]
enabled = true
endpoint = "https://your-collector.example.com/memscope-events"
```

Both fields are required — `enabled = true` without an endpoint puts
MemScope in `opt-in-misconfigured` mode and emits nothing.

### How to disable

Either omit the `[telemetry]` block entirely (default state) or set:

```toml
[telemetry]
enabled = false
```

You can confirm the state by inspecting the JSON report:
`diagnostics.telemetry.mode` will be `off`, `opt-in-active`, or
`opt-in-misconfigured`.

### Operational guarantees

- Telemetry POSTs are **best-effort with a 1-second timeout**. A slow or
  unreachable endpoint cannot slow down or break MemScope.
- Network errors, timeouts, and HTTP 4xx/5xx responses from the endpoint
  are dropped silently. The CLI exit code reflects the analysis result,
  never a telemetry failure.
- Telemetry runs **after** the command completes — the payload is
  recorded only when you can already see the analysis output.

## Local data files MemScope writes

MemScope stores small state files under `~/.memscope/`:

| File | Purpose | What's in it | Written by |
|---|---|---|---|
| `eula_state.json` | EULA acceptance record | Acceptance timestamp, EULA hash | Free core |
| `install_id` | Stable per-install UUID | A UUIDv4 generated on first run | Free core (only used by opt-in telemetry, if enabled) |
| `license_cache.json` | Cached activation receipt | License Key, machine fingerprint hash, expires_at, HMAC signature | Subscription Bundles only — created on Bundle activation |

The free core writes only `eula_state.json` and `install_id`.
The `license_cache.json` is written only after a Subscription Bundle is
activated; users who never install a Subscription Bundle will never see
this file.

These files never leave your machine unless you ship them yourself.
Deleting them (`rm -rf ~/.memscope/`) resets MemScope to the
"never-installed" state.

## Reports MemScope writes

Reports (HTML, JSON, CSV) are written wherever you tell MemScope to write
them via the `--html` / `--json` / `--csv` flags or the `[reports]`
section of your config. They contain analysis output from the firmware
artifacts you provided. **MemScope does not transmit reports anywhere** —
they are local files, owned by you.

The HTML report includes a small license-attestation footer (license id
last 4 chars, expiry date, mode) so a customer's compliance team can tell
which license issued the report.

## Auditing what we just said

You can audit MemScope's network behavior at runtime with:

```bash
# On Linux/macOS:
strace -e trace=network -f memscope analyze ...

# Or run under a network firewall that denies everything by default and
# observe what gets blocked.
```

The only outbound connections you should ever see (without telemetry
opted in) are to your configured license backend (`api.keygen.sh` for
the default Keygen integration), and only during license-related
operations.
