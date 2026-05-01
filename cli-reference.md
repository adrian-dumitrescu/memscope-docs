# MemScope CLI Reference

This reference documents the `memscope` command surface for the
`memscope-fw` core distribution on PyPI. The core is free for any use,
including commercial use; no license key, no watermark, no time limit.

## Global usage

```bash
memscope [GLOBAL OPTIONS] COMMAND [COMMAND OPTIONS]
```

### Global options

- `--config TEXT`: path to `memscope.toml`.
- `--log-level TEXT`: `error|warning|info|debug|trace` (default `info`).
- `--strict`: enables strict failure behavior (non-zero exit on policy violations).
- `--toolchain TEXT`: toolchain hint override.
- `--target-name TEXT`: target name override.
- `--version`: prints version and exits.

### Exit codes

- `0`: success.
- `1`: policy failure (e.g. budget violation in `--strict` mode).
- `2`: input/parsing/config failure.
- `3`: license failure (e.g. EULA not accepted; rare in normal use).

## Commands

### `analyze`

Analyze one build and generate summary outputs.

```bash
memscope analyze [--elf PATH] [--map PATH] [--linker PATH] [--json PATH] [--html PATH] [--csv PATH] [--baseline-json PATH]
```

Options:

- `--elf PATH`: input ELF file.
- `--map PATH`: input MAP file.
- `--linker PATH`: linker script/config.
- `--json PATH`: JSON report output.
- `--html PATH`: HTML report output.
- `--csv PATH`: CSV report output.
- `--baseline-json PATH`: baseline JSON report for diff/regression analysis.

### `validate`

Validate config, artifact discovery, parser compatibility, and normalized model generation. Intended as a CI-pipeline sanity check that runs on every build.

```bash
memscope validate [--elf PATH] [--map PATH] [--linker PATH]
```

Options:

- `--elf PATH`
- `--map PATH`
- `--linker PATH`

### `diff`

Compare two MemScope JSON reports.

```bash
memscope diff --current-json PATH --baseline-json PATH [--json PATH] [--html PATH]
```

Options:

- `--current-json PATH`: current report (required).
- `--baseline-json PATH`: baseline report (required).
- `--json PATH`: optional diff JSON output.
- `--html PATH`: optional diff HTML output.

### `accept-terms`

Record EULA acceptance non-interactively. Useful in CI / scripted setup where stdin is not a TTY and the interactive EULA prompt cannot fire.

```bash
memscope accept-terms
```

### `diagnostics export`

Generate a sanitized diagnostics ZIP for support requests. Includes parser diagnostics, environment metadata, configuration snapshot, and any errors encountered — paths and identifiers are redacted before bundling.

```bash
memscope diagnostics export [--output PATH]
```

Options:

- `--output PATH`: diagnostics zip output path (default `diagnostics.zip`).

### `dev inspect`

Developer inspection commands. Useful for debugging unusual parser behaviour or contributing to MemScope.

```bash
memscope dev inspect parse [--elf PATH] [--map PATH] [--linker PATH] [--output PATH]
memscope dev inspect model [--output PATH]
memscope dev inspect timings [--output PATH]
memscope dev inspect rules [--output PATH]
```

## Behavior notes

- MemScope enforces local-only path policy and rejects remote/network artifact or config references (URLs / UNC paths). All analysis runs locally.
- The free core makes no network calls in normal operation. No telemetry, no update checks, no license server contact. Telemetry is opt-in only via `[telemetry]` in `memscope.toml`.
- The first run after upgrade may prompt for EULA acceptance if the EULA hash has changed; on non-TTY contexts (CI), run `memscope accept-terms` once to record acceptance.

## Optional commercial Bundles

The free core listed above is everything available via `pip install memscope-fw`. Optional commercial Bundles (PR / Slack notifications, SoC-family analysis extras, ISO 26262 / IEC 62304 / DO-178C compliance evidence packages, custom toolchain support, bespoke per-customer features) are delivered as Customer-Bound Wheels separately from PyPI under purchase contracts. Bundle licences are not yet generally available for sale — see the project README for status.
