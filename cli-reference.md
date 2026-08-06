# MemScope CLI Reference

This reference documents the `memscope` command surface for the
`memscope-fw` distribution on PyPI. MemScope is free for any use,
including commercial use; no license key, no watermark, no time limit,
and no command locked behind a purchase — everything documented here
is in the distribution you install.

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
memscope analyze [--elf PATH] [--map PATH] [--linker PATH] \
                 [--json PATH] [--html PATH] [--csv PATH] \
                 [--text PATH] [--markdown PATH] [--top-symbols N] \
                 [--baseline-json PATH]
```

Options:

- `--elf PATH`: input ELF file.
- `--map PATH`: input MAP file.
- `--linker PATH`: linker script/config.
- `--json PATH`: JSON report output.
- `--html PATH`: HTML report output.
- `--csv PATH`: CSV report output.
- `--text PATH`: plain-text memory layout dump (see [Memory layout dumps](#memory-layout-dumps)).
- `--markdown PATH`: markdown memory layout dump (see [Memory layout dumps](#memory-layout-dumps)).
- `--top-symbols N`: per-section symbol cap for `--text` / `--markdown` dumps (default `5`; `0` = show every symbol, no cap). Equivalent TOML key: `reports.layout_top_symbols`.
- `--baseline-json PATH`: baseline JSON report for diff/regression analysis.

#### Memory layout dumps

`--text` and `--markdown` produce a hierarchical, **full-memory** dump of the
Memory Layout Explorer — every region, every section in address order, and the
top 5 largest symbols per section with their addresses, sizes, percentages, and
owning object file. Designed for PR comments, build logs, and `diff`-friendly
snapshots that JSON and CSV can't provide.

Example:

```bash
memscope analyze --map build/firmware.map --linker build/firmware.ld \
                 --text  reports/layout.txt \
                 --markdown reports/layout.md
```

**Controlling how many symbols appear per section.** By default, each section
lists its top 5 largest symbols followed by a `... and N more symbols` hint.
Tune the cap with `--top-symbols N` (or `reports.layout_top_symbols = N` in
TOML):

```bash
# Show all symbols, no cap (useful for full auditability or diff workflows).
memscope analyze --map build/firmware.map --linker build/firmware.ld \
                 --text reports/layout.txt --top-symbols 0

# Show the top 20 per section.
memscope analyze --map build/firmware.map --linker build/firmware.ld \
                 --markdown reports/layout.md --top-symbols 20
```

The cap is independent of the HTML side panel's display (which always shows up
to 24 entries). Setting `--top-symbols 0` walks the full symbol table, so
sections with hundreds of symbols are all rendered — file size grows linearly
with the total symbol count.

The plain variant uses column alignment:

```text
================================================================================
Region: FLASH                                  34.6%       725,632 /    2,097,152 B
        0x08000000 – 0x081FFFFF
================================================================================

  .text                                kind=section  size=     720,128 B ( 34.3%) 0x08000400 – 0x080AFEFF
      motorCtrl_init                          5,184 B   0x08000400  (motor_ctrl.o)
      pid_update                              3,840 B   0x08001840  (pid.o)
      ...
```

The markdown variant uses `## ` headings + fenced code blocks per section,
renders cleanly in GitHub / GitLab / chat clients, and stays diff-friendly.

Both dumps cover the **entire** memory layout regardless of any HTML-side view
state — they don't honor the canvas zoom, Focus dropdown, or Group-by mode (those
only affect the interactive HTML report). For the equivalent in-browser exports
that copy the same content to clipboard or save as files, see the
HTML report's Share menu.

#### Multi-core analysis

When your firmware ships as N ELFs (one per core) sharing a single physical
memory map, use `--core` to produce a single combined report covering all
cores instead of analyzing each separately.

**Spec.** `--core NAME=ELF[,MAP[,LINKER]]`. The `NAME` is your label for the
core (e.g., `core0`); the path triplet is comma-separated. ELF is required;
MAP and LINKER are optional. Repeat `--core` once per core.

`--core` is **mutually exclusive** with `--elf` / `--map` / `--linker`. Mixing
them is a CLI error.

**Worked example** (3-core S32K396 firmware project):

```bash
memscope analyze \
  --core core0=Inverter_core0.elf,Inverter_core0.map,linker_flash_c0_s32k396.ld \
  --core core1=DC-DC_core1.elf,DC-DC_core1.map,linker_flash_c1_s32k396.ld \
  --core core2=SSM_core2.elf,SSM_core2.map \
  --html combined.html
```

`core2` in this example has no linker — the canonical memory map comes from
`core0` + `core1`'s linkers, which together describe the full chip. You only
need linkers from enough cores to cover all relevant regions.

Console output confirms detection:

```text
Cores Detected: 3 (core0, core1, core2)
```

**How it works.**

1. Each `--core` is parsed and normalized independently.
2. All linker `MEMORY{}` blocks are reconciled into one canonical memory map. Identical declarations merge silently; partial overlap (region defined in some linkers, not others) keeps the defined version; attribute differences (e.g., `(rx)` vs `(rwx)`) are tolerated; only `ORIGIN`/`LENGTH` disagreement raises a clear error listing every disagreement per linker.
3. Per-core sections and symbols are unioned into a single model, tagged with their `core_id`. Section-name collisions across cores (e.g., each core has its own `.text`) are preserved — they have distinct VMAs.
4. The merged model feeds the existing analysis pipeline. The report's `summary.extra.multicore` block lists the declared cores; sections and symbols carry a `core_id` field in the JSON output for downstream tools.

**Limitation.** When two or more cores have sections at the same VMA inside
the same canonical region (the classic case is per-core TCMs at address
`0x0`), MemScope emits a `multicore_warning` diagnostic but does not split
the region in this release. A future MemScope release will add an explicit
TOML config block letting you split such aliased regions into per-core
virtual entries.

##### Region-name heuristic detection (and how to disable it)

In addition to explicit `--core` invocations, MemScope also infers multicore
from region naming patterns: a single linker script whose `MEMORY{}` block
declares regions like `core0_flash` / `core1_flash`, `cm4_*` / `cm7_*`, or
`int_flash_c0` / `int_flash_c1` is auto-detected as multicore even with a
single `--map`/`--linker` invocation. This is convenient for single-ELF
firmware on SoCs whose linker scripts naturally segregate per-core regions.

**It can also be wrong.** Some SoC vendor linker scripts statically declare
*all* per-core regions even when the build only populates one (e.g. an
STM32H7 dual-core script that names both `cm4_*` and `cm7_*` regions for
hardware addressing reasons, used on a single-image firmware that only
runs on the CM7). Pass `--no-infer-multicore` to disable the heuristic and
require explicit `--core` / `[[inputs.cores]]` for multicore detection:

```bash
memscope analyze \
  --map firmware.map --linker firmware.ld \
  --no-infer-multicore \
  --html report.html
```

Explicit `--core` and `[[inputs.cores]]` invocations always declare
multicore regardless of `--no-infer-multicore` — the flag only disables
the region-name heuristic on single-image runs. For the persistent TOML
form, see
[`configuration-reference.md` § Multi-core analysis](configuration-reference.md#multi-core-analysis).

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
- MemScope makes no network calls in normal operation. No telemetry, no update checks, no licensing backend. Telemetry is opt-in only via `[telemetry]` in `memscope.toml`.
- The first run after upgrade may prompt for terms acceptance if the terms text has changed; on non-TTY contexts (CI), run `memscope accept-terms` once to record acceptance.

## Custom work

The commands above are the complete surface of `pip install memscope-fw`
— there is no additional paid command set. If you need something
MemScope doesn't support (a toolchain or proprietary linker format it
can't parse, an adapter for an internal build format), that is arranged
separately as its own software under its own written agreement; see
clause 3 of the LICENSE. Enquiries: <dumitrescu.adrian121@gmail.com>.
