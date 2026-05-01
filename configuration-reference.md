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
json = "build/reports/app.json"
html = "build/reports/app.html"
csv = "build/reports/app.csv"
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
emit_command_usage = false
emit_crash_reports = false
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

## `[reports]`

- `json` (`string | null`): JSON output path.
- `html` (`string | null`): HTML output path.
- `csv` (`string | null`): CSV output path.
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
- `emit_command_usage` (`bool`, default `false`)
- `emit_crash_reports` (`bool`, default `false`)

Current behavior:

- Telemetry is disabled by default.
- The v1 telemetry path is a future-safe opt-in stub.
- If `enabled=true`, an endpoint must be provided.

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
