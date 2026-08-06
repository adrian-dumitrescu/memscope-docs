# MemScope Configuration Reference

MemScope loads configuration in this order:

1. Built-in defaults
2. Config file (`--config` or discovered `memscope.toml` / `.memscope.toml` / `memscope.config.toml`)
3. CLI overrides

## Full example

```toml
[project]
name = "app"
toolchain = "gnu"

[inputs]
elf = "build/app.elf"
map = "build/app.map"
linker = "linker.ld"

[reports]
json     = "build/reports/app.json"
html     = "build/reports/app.html"
csv      = "build/reports/app.csv"
text     = "build/reports/app-layout.txt"
markdown = "build/reports/app-layout.md"
layout_top_symbols = 5     # 0 = show all symbols, no cap
enable_sankey_view = false
enable_relationship_graph_view = false

[budgets.flash]
FLASH = 0.90

[budgets.ram]
SRAM = 0.85

[policies]
fail_on_overflow = true
fail_on_reserved_region_violation = true
warn_on_large_bss_symbol_kb = 8
warn_on_near_capacity_ratio = 0.90
max_ram_growth_bytes = 2048
max_flash_growth_bytes = 4096
ota_source_regions = ["APP_FLASH"]
ota_slot_a_max_bytes = 262144
ota_slot_b_max_bytes = 262144
ota_warn_ratio = 0.90
dma_section_patterns = [".dma_*"]
dma_safe_regions = ["DMA_RAM"]
non_cacheable_section_patterns = [".nocache_*"]
non_cacheable_regions = ["SRAM_NOCACHE"]
cacheable_section_patterns = [".cache_*"]
cacheable_regions = ["SRAM_CACHE"]
retention_section_patterns = [".retain_*"]
retention_regions = ["RET_RAM"]
suppress_issue_ids = []
suppress_issue_prefixes = []
suppress_categories = []

[policies.partition_max_bytes]
BOOT_FLASH = 65536
APP_FLASH = 262144

[policies.forbidden_component_regions]
bootloader = ["SRAM", "DTCM"]

[groups]
Platform = ["platform/*", "hal/*"]
Application = ["app/*", "services/*"]

[telemetry]
enabled = false
```

## Section reference

## `[project]`

- `name` (`string | null`): target name displayed in reports.
- `toolchain` (`string | null`): hint (`gnu`, `clang_lld`, `iar`, `keil`, etc.).

## `[inputs]`

- `elf` (`string | null`): ELF path.
- `map` (`string | null`): MAP path.
- `linker` (`string | null`): linker script/config path.

All paths must be local paths; remote/network references are rejected.

### Multi-core analysis

When your firmware ships as N ELFs (one per core) sharing a single physical
memory map, use `[[inputs.cores]]` instead of `inputs.elf`/`inputs.map`/`inputs.linker`.
MemScope produces one combined report covering all cores. The CLI form
`--core NAME=ELF[,MAP[,LINKER]]` is the per-core equivalent — see
[`cli-reference.md` § Multi-core analysis](cli-reference.md#multi-core-analysis).
When both CLI and TOML are supplied, CLI wins silently.

**`[[inputs.cores]]`** — array-of-tables, one entry per core. Each entry:

- `name` (`string`, required): your label for the core (e.g., `core0`).
- `elf` (`string | null`): per-core ELF path.
- `map` (`string | null`): per-core MAP path.
- `linker` (`string | null`): per-core linker script. Optional; some cores in
  a multi-core build (commonly the "data-only" cores) ship without their own
  linker — MemScope synthesizes the canonical memory map from the cores that
  do provide one.

**`inputs.canonical_linker`** (`string | null`) — rare override. When per-core
linkers disagree on a region's origin/length, MemScope normally raises a
`LinkerConflictError` with structured diagnostics. Set this to force one
specific linker as authoritative and bypass reconciliation.

**`[inputs.private_regions]`** — region-name → `{core_name: virtual_suffix}` mapping.
When two or more cores have sections at the same VMA inside the same
canonical region (the classic case is per-core TCMs at `0x0`), MemScope splits
the region into per-core virtual entries (`int_itcm@core0`, `int_itcm@core1`, …).
Without this block, MemScope emits a `multicore_warning` and prints a
copy-paste-ready TOML snippet showing exactly what to add here — most users
let MemScope auto-generate the suggestion the first time and paste it in.

**Worked example** — S32K396 3-core firmware project:

```toml
[project]
name = "s32k396-3core"

[[inputs.cores]]
name = "core0"
elf = "build/Inverter_core0.elf"
map = "build/Inverter_core0.map"
linker = "linker_flash_c0_s32k396.ld"

[[inputs.cores]]
name = "core1"
elf = "build/DC-DC_core1.elf"
map = "build/DC-DC_core1.map"
linker = "linker_flash_c1_s32k396.ld"

[[inputs.cores]]
name = "core2"
elf = "build/SSM_core2.elf"
map = "build/SSM_core2.map"

[inputs.private_regions]
int_itcm = { core0 = "core0", core1 = "core1", core2 = "core2" }
int_dtcm = { core0 = "core0", core1 = "core1", core2 = "core2" }

[reports]
html = "build/reports/combined.html"
json = "build/reports/combined.json"
```

`core2` in this example has no linker — the canonical memory map comes from
`core0` + `core1`'s linkers, which together describe the full chip. The
`[inputs.private_regions]` block splits the per-core TCMs (`int_itcm`,
`int_dtcm`) into three virtual entries each so per-core symbols attribute
correctly. Run with no `--core` flags: `memscope analyze --config memscope.toml`.

**`inputs.infer_multicore_from_region_names`** (`bool`, default `true`) —
toggle for the region-name heuristic multicore detection. When `true`
(default), single-image builds whose `MEMORY{}` block declares regions
like `core0_flash` / `core1_flash`, `cm4_*` / `cm7_*`, or
`int_flash_c0` / `int_flash_c1` are auto-detected as multicore. Set to
`false` to disable the heuristic for SoCs whose vendor linker scripts
statically declare all per-core regions even on single-image firmware
(STM32H7 dual-core variants populated as single-core, etc.). The CLI
equivalent is `--no-infer-multicore`; CLI wins over TOML when both are
set. Explicit `[[inputs.cores]]` always declares multicore regardless of
this setting — it only controls the heuristic on single-image runs.

```toml
[inputs]
map = "build/firmware.map"
linker = "vendor_dual_core.ld"
infer_multicore_from_region_names = false  # single-image despite cm4_/cm7_ regions
```

## `[reports]`

- `json` (`string | null`): JSON output path.
- `html` (`string | null`): HTML output path.
- `csv` (`string | null`): CSV output path.
- `text` (`string | null`): plain-text memory layout dump (full-memory hierarchical region → section → top symbols tree). See [Memory layout dumps](cli-reference.md#memory-layout-dumps) in the CLI reference for the format.
- `markdown` (`string | null`): markdown memory layout dump (same content as `text`, renders cleanly in GitHub / GitLab / chat clients and PR comments).
- `layout_top_symbols` (`int | null`, default `null` → uses built-in 5): per-section symbol cap for `text` / `markdown` dumps. Set to `0` to show every symbol with no cap. CLI override: `--top-symbols N`.
- `enable_sankey_view` (`bool`, default `false`): enables optional `input section -> output section -> memory region` Sankey view in HTML.
- `enable_relationship_graph_view` (`bool`, default `false`): enables optional advanced relationship graph tab in HTML.

## `[budgets.flash]` and `[budgets.ram]`

Region-to-ratio maps:

- key: region name (`FLASH`, `SRAM`, etc.)
- value: target utilization ratio (for example `0.9`)

## `[policies]`

- `fail_on_overflow` (`bool`, default `true`)
- `fail_on_reserved_region_violation` (`bool`, default `true`)
- `warn_on_large_bss_symbol_kb` (`int | null`, default `8`)
- `warn_on_near_capacity_ratio` (`float | null`, default `0.9`)
- `max_ram_growth_bytes` (`int | null`, default `null`)
- `max_flash_growth_bytes` (`int | null`, default `null`)
- `partition_max_bytes` (`table[str,int]`, default `{}`)
- `ota_source_regions` (`list[str]`, default `[]`)
- `ota_slot_a_max_bytes` (`int | null`, default `null`)
- `ota_slot_b_max_bytes` (`int | null`, default `null`)
- `ota_warn_ratio` (`float | null`, default `0.9`)
- `dma_section_patterns` (`list[str]`, default `[]`)
- `dma_safe_regions` (`list[str]`, default `[]`)
- `non_cacheable_section_patterns` (`list[str]`, default `[]`)
- `non_cacheable_regions` (`list[str]`, default `[]`)
- `cacheable_section_patterns` (`list[str]`, default `[]`)
- `cacheable_regions` (`list[str]`, default `[]`)
- `retention_section_patterns` (`list[str]`, default `[]`)
- `retention_regions` (`list[str]`, default `[]`)
- `suppress_issue_ids` (`list[str]`, default `[]`)
- `suppress_issue_prefixes` (`list[str]`, default `[]`)
- `suppress_categories` (`list[str]`, default `[]`)

`[policies.forbidden_component_regions]` maps component names to forbidden region lists.
`[policies.partition_max_bytes]` maps memory region names to hard byte limits.

## `[groups]`

Component grouping map:

- key: group name
- value: symbol/object glob-like patterns

Pattern behavior:

- groups are evaluated in declaration order (first match wins)
- patterns are case-insensitive
- plain patterns match across ownership fields (`object`, `archive`, `compilation unit`, `source`, `symbol`, `section`)
- scoped patterns are supported:
  - `object:<glob>`
  - `archive:<glob>`
  - `compilation_unit:<glob>` (or `unit:<glob>`)
  - `source:<glob>`
  - `symbol:<glob>`
  - `section:<glob>`

When no user group matches, MemScope applies default heuristic tags (`Drivers`, `RTOS`, `Middleware`, `Startup`, `Application`) before falling back to `Unassigned`.

## `[telemetry]`

- `enabled` (`bool`, default `false`)
- `endpoint` (`string | null`, default `null`)

Current behavior:

- Telemetry is disabled by default; omitting the `[telemetry]` block
  entirely means no request is ever issued.
- If `enabled = true`, an `endpoint` **must** be provided — otherwise
  config loading fails with a validation error rather than silently
  doing nothing.
- When enabled, MemScope POSTs exactly `version`, `command`, and
  `runtime_ms` to the endpoint you specify, after the command
  completes. A whitelist enforced in code drops any other field.
- Best-effort with a 1-second timeout; network errors and non-2xx
  responses are dropped silently and never affect the exit code.
- The Licensor neither receives nor stores telemetry data — you choose
  the recipient and you are its data controller. See
  [`privacy.md`](privacy.md).

## Policy recipes

## Strict CI gating

```toml
[policies]
fail_on_overflow = true
fail_on_reserved_region_violation = true
warn_on_near_capacity_ratio = 0.85
max_ram_growth_bytes = 0
max_flash_growth_bytes = 0
```

Run with:

```bash
memscope --strict analyze --config memscope.toml
```

## Allow intentional known issue IDs

```toml
[policies]
suppress_issue_ids = ["missing_heap_definition"]
```

## Guard component placement

```toml
[policies.forbidden_component_regions]
crypto = ["SRAM"]
```

## Enforce DMA/cacheability/retention placement

```toml
[policies]
dma_section_patterns = [".dma_*"]
dma_safe_regions = ["DMA_RAM"]
non_cacheable_section_patterns = [".nocache_*"]
non_cacheable_regions = ["SRAM_NOCACHE"]
cacheable_section_patterns = [".cache_*"]
cacheable_regions = ["SRAM_CACHE"]
retention_section_patterns = [".retain_*"]
retention_regions = ["RET_RAM"]
```
