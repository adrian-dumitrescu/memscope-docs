# MemScope Troubleshooting

This guide covers common setup and parsing problems and the fastest fix path.

## Command fails with exit code `2`

Exit code `2` means input/parsing/config failure.

Checklist:

1. Confirm files exist (`--elf`, `--map`, `--linker`).
2. Run `memscope validate ...` first to isolate parse/config issues.
3. Confirm config file path is correct (`--config`).

## "Remote/network path is not allowed in local-only mode"

MemScope rejects UNC paths and URL-like references by design.

Fix:

1. Copy required artifacts/config locally.
2. Pass local filesystem paths only.

## Config validation errors

Common causes:

- wrong type (`bool` expected but string provided)
- unknown section/key
- empty string values where non-empty strings are required
- `telemetry.enabled=true` without `telemetry.endpoint`

Fix:

1. Start from [configuration-reference.md](configuration-reference.md).
2. Run `memscope validate --config memscope.toml`.

## Parser diagnostics warning about ambiguity

Typical messages:

- `crosscheck.region_set_mismatch`
- `crosscheck.unknown_execution_region`
- `gnu_map.section_region_ambiguity`

What to do:

1. Ensure MAP and linker script come from the same build.
2. Ensure linker region names are consistent.
3. Use `--toolchain` if auto-detection is uncertain.
4. Inspect parse output with `memscope dev inspect parse`.

## Missing stack/heap definition warnings

Issue IDs:

- `missing_stack_definition`
- `missing_heap_definition`

These are often expected when stack/heap markers are not explicit.

Options:

1. Add explicit stack/heap linker symbols/sections.
2. Suppress with policy if intentional.

## `diff` reports schema mismatch

Cause:

- baseline and current JSON reports are not compatible versions.

Fix:

1. Regenerate baseline with the same MemScope version.
2. Keep stable golden/baseline reports in source control.

## EULA-acceptance failures (exit code `3`)

The free core requires one-time EULA acceptance. On a TTY, MemScope
prompts on first run; on non-interactive shells (CI / scripts) the
prompt cannot fire and exit code `3` is returned with a hint.

Fix:

1. Run `memscope accept-terms` once from a terminal (or any environment
   with stdin connected) to record acceptance.
2. Confirm `~/.memscope/eula_state.json` is writable. The state file is
   small (< 1 KB) and must persist between runs.
3. If the EULA hash has changed (after upgrading MemScope to a new
   major version), re-run `accept-terms` to record the new acceptance.

## Need support-ready context

Generate a sanitized diagnostics bundle:

```bash
memscope diagnostics export --output diagnostics.zip
```

Then follow [support-workflow.md](../internal/handbook/support-workflow.md).
