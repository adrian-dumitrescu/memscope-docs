# MemScope privacy and telemetry policy

> **Effective:** 2026-08-06 — applies to MemScope (`memscope-fw` on PyPI)
> v0.2.0a13 and later.
> **Data controller:** none — MemScope transmits no data to the
> Licensor, so the Licensor acts as neither controller nor processor
> for your use of it (see GDPR posture below).
> **Publisher:** Adrian Dumitrescu (Romania), to be assigned to the
> Licensor's commercial entity upon registration.
> **Contact for privacy questions:** dumitrescu.adrian121@gmail.com

MemScope is a **local-first** tool: it runs against your firmware artifacts
on your machine, writes reports to your local filesystem, and **does not
phone home or transmit any data by default**.

MemScope makes **no network calls of any kind** in normal operation.
There is no licensing backend, no activation, no licence check, no
update check, and no usage reporting. Anyone can install it, run
`memscope analyze`, and never have a single byte leave their machine.

The **only** situation in which MemScope contacts the network is
**opt-in telemetry**, described below. It is **OFF by default** and
requires you to explicitly configure both a flag and an endpoint that
you choose. The Licensor does not receive that data.

## GDPR posture

For users in the European Economic Area, the United Kingdom, and other
jurisdictions with comparable data-protection regimes:

- **MemScope (everyone):** MemScope makes no network calls and
  processes no personal data on the Licensor's behalf. **The Licensor
  is neither a data controller nor a data processor with respect to
  your use of MemScope.** No personal data is collected, transmitted,
  or stored by the Licensor.
- **No sub-processors.** MemScope uses none. There is no licensing
  backend, no analytics provider, and no third party receiving any
  data about your use of the software.
- **Opt-in telemetry:** when you enable telemetry by setting
  `[telemetry].endpoint` in your `memscope.toml`, the **endpoint you
  configure** is the recipient. The Licensor neither receives nor
  stores telemetry data — it only ships the code that POSTs to the
  endpoint of your choosing. **You** are the data controller for any
  data sent to your own collector.
- **Data-subject rights:** because the Licensor processes no personal
  data in connection with MemScope, there is no controller-side data
  to access, rectify, erase, restrict, or port. Enquiries are welcome
  regardless: dumitrescu.adrian121@gmail.com. The Licensor responds
  within 30 days.
- **Custom Extensions:** if the Licensor ever supplies a User with
  custom software under a separate written agreement (clause 3 of the
  LICENSE), that software is not MemScope, is not covered by this
  policy, and comes with its own privacy terms and its own stated
  lawful basis for any processing it performs.

## Opt-in telemetry

If you enable telemetry, MemScope reports a tiny payload to an HTTPS
endpoint of YOUR choosing (you control where the data goes):

### What is collected (and only this)

| Field | Example | What it tells us |
|---|---|---|
| `version` | `0.2.0a13` | Which release of MemScope is in use |
| `command` | `analyze` | Which subcommand ran (analyze, diff, validate) |
| `runtime_ms` | `4521` | How long the command took, in milliseconds |

That is the complete payload. The whitelist enforced in code also
permits `exit_code`, which is reserved and not currently sent; nothing
outside that whitelist can be transmitted even by a future caller.

### What is NEVER collected

- File paths (ELF, MAP, linker, output paths).
- Target name (project / firmware identifier).
- Toolchain identifier.
- Memory region names, section names, symbol names.
- Issue counts, byte counts, or any analysis output.
- Hostname, IP address, MAC address, or other network identifiers.
- Any machine, host, or install identifier.
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
| `eula_state.json` | Terms-acceptance record | Acceptance timestamp, hash of the accepted text | MemScope, on first run |
| `install_id` | Stable per-install UUID | A UUIDv4 generated on first run | Written only if a component that needs a local install identifier runs; it is **never transmitted** — it is not in the telemetry whitelist |

Neither file contains personal data beyond what your own machine
already holds, and neither is transmitted anywhere.

These files never leave your machine unless you ship them yourself.
Deleting them (`rm -rf ~/.memscope/`) resets MemScope to the
"never-installed" state.

## Reports MemScope writes

Reports (HTML, JSON, CSV) are written wherever you tell MemScope to write
them via the `--html` / `--json` / `--csv` flags or the `[reports]`
section of your config. They contain analysis output from the firmware
artifacts you provided. **MemScope does not transmit reports anywhere** —
they are local files, owned by you.

Reports carry no licence identifiers, no watermarks, and no
attestation footer. They contain your analysis output and nothing
about you or your licence.

## Auditing what we just said

You can audit MemScope's network behavior at runtime with:

```bash
# On Linux/macOS:
strace -e trace=network -f memscope analyze ...

# Or run under a network firewall that denies everything by default and
# observe what gets blocked.
```

Without telemetry opted in, you should see **no outbound connections at
all**. If you see any, that is a bug — please report it at the contact
address above.

With telemetry enabled, the only connection is a POST to the endpoint
you configured yourself in `memscope.toml`.
