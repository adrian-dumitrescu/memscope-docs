# MemScope Configuration Guide

This is the user-facing companion to [`configuration-reference.md`](configuration-reference.md).
The reference lists every field with its type and default. **This guide
explains how each rule actually works**, what it produces in the report,
and shows complete worked examples per use case.

## Table of contents

- [How configuration is loaded](#how-configuration-is-loaded)
- [Top-level sections](#top-level-sections)
- [`[project]` — report identity](#project--report-identity)
- [`[inputs]` — default file paths](#inputs--default-file-paths)
- [`[reports]` — output paths and optional views](#reports--output-paths-and-optional-views)
- [`[budgets]` — soft per-region capacity caps](#budgets--soft-per-region-capacity-caps)
- [`[policies]` — gates and warnings](#policies--gates-and-warnings)
  - [Overflow and near-capacity](#overflow-and-near-capacity)
  - [Large BSS symbol warnings](#large-bss-symbol-warnings)
  - [Hard byte caps with `partition_max_bytes`](#hard-byte-caps-with-partition_max_bytes)
  - [OTA fit gates](#ota-fit-gates)
  - [Placement gates: DMA / cacheable / non-cacheable / retention](#placement-gates-dma--cacheable--non-cacheable--retention)
  - [Component placement: `forbidden_component_regions`](#component-placement-forbidden_component_regions)
  - [Build-to-build regression gates](#build-to-build-regression-gates)
  - [Suppression](#suppression)
  - [Stack and heap detection](#stack-and-heap-detection)
- [`[groups]` — collapse symbols into logical components](#groups--collapse-symbols-into-logical-components)
- [`[telemetry]` — opt-in usage data](#telemetry--opt-in-usage-data)
- [End-to-end recipes](#end-to-end-recipes)
- [Issue ID reference](#issue-id-reference)

---

## How configuration is loaded

MemScope merges three layers, last-wins:

1. **Built-in defaults** (the field defaults documented in [`configuration-reference.md`](configuration-reference.md))
2. **TOML file** — explicit `--config <path>` or auto-discovered (looks for
   `memscope.toml`, `.memscope.toml`, `memscope.config.toml` in the
   current working directory)
3. **CLI flags** (`--elf`, `--map`, `--linker`, `--baseline-json`, etc.)

`--config` is a **top-level option on `memscope`**, not a flag on
`analyze`. Order matters:

```bash
# CORRECT
memscope --config memscope.toml analyze --elf app.elf --map app.map

# WRONG — analyze rejects --config as an unknown option
memscope analyze --config memscope.toml --elf app.elf
```

Verify your config is being applied by re-running with and without `--config`,
then diffing the JSON `issues` array. If they're identical, your TOML is
either empty, has no rules tighter than the defaults, or doesn't match
your project's actual region/section names.

---

## Top-level sections

| Section | Purpose | Example |
|---|---|---|
| `[project]` | Cosmetic — name shown in the report header | `name = "my_app"` |
| `[inputs]` | Default ELF/MAP/linker paths so you don't repeat them on the CLI | `elf = "build/app.elf"` |
| `[reports]` | Default JSON/HTML/CSV output paths | `html = "reports/app.html"` |
| `[budgets]` | Per-region utilization ratios that trigger `budget.<region>` warnings | `[budgets.flash]\nFLASH = 0.85` |
| `[policies]` | All gates: overflow, partition caps, OTA, DMA, suppression, regression, etc. | see below |
| `[groups]` | Map source-path or object-file globs to logical component names | `MCAL = ["**/Mcal*.o"]` |
| `[telemetry]` | Opt-in anonymous usage data | `enabled = false` (default) |

---

## `[project]` — report identity

```toml
[project]
name = "my_firmware"        # appears in the HTML report header
toolchain = "gnu"           # optional hint: gnu / clang_lld / iar / keil
```

Cosmetic only — does not affect issue evaluation.

---

## `[inputs]` — default file paths

```toml
[inputs]
elf    = "build/app.elf"
map    = "build/app.map"
linker = "build/app.ld"
```

Paths are relative to the **current working directory**, not the TOML
file. CLI flags override (`--elf path/here.elf` wins over the TOML's
`elf` value).

Useful when you want a single command:
```bash
memscope --config memscope.toml analyze
```
…instead of:
```bash
memscope analyze --elf build/app.elf --map build/app.map --linker build/app.ld
```

---

## `[reports]` — output paths and optional views

```toml
[reports]
json = "reports/app.json"
html = "reports/app.html"
csv  = "reports/app.csv"
enable_sankey_view = false              # extra HTML tab: input → output → region
enable_relationship_graph_view = false  # extra HTML tab: symbol relationship graph
```

Same path-override rule as `[inputs]`: CLI `--json`/`--html`/`--csv` wins.

The two `enable_*_view` flags are off by default because the views are
heavy on browser memory for very large maps. Turn them on selectively
if you want them in the rendered HTML.

---

## `[budgets]` — soft per-region capacity caps

Two sub-tables: `[budgets.flash]` and `[budgets.ram]`. Both take the
same shape — `region_name = ratio` — and the analyzer treats them
identically. The split is just for organization (lets you see flash
budgets and RAM budgets at a glance).

```toml
[budgets.flash]
FLASH      = 0.85              # warn when FLASH region exceeds 85% used
APP_FLASH  = 0.90

[budgets.ram]
SRAM       = 0.80
DTCM       = 0.85
```

**Effect.** For each entry, MemScope checks the region's actual
utilization. If `actual > budget`, a `budget.<region>` warning appears
in the **Issues and Recommendations** section.

**Interaction with `near_capacity`.** Budgets and `near_capacity` are
independent gates. If a region has *both* a budget AND its util crosses
`warn_on_near_capacity_ratio`, only the budget warning fires (it's
treated as the more specific signal). If you set a budget tighter than
`near_capacity`, you get earlier warnings with region-specific labels.

**Typo detection.** If you put a budget key for a region that doesn't
exist in the linker (typo, stale rename, or only in a different build
variant), MemScope emits an `unknown_budget_region.<name>` warning.
Without that warning the budget would silently do nothing — easy to
miss, and exactly the case where CI thinks it's protected and isn't.

**Worked example** (S32K324):

```toml
[budgets.flash]
int_pflash_region = 0.85       # 1 MB flash, warn at 850 KB
int_dflash_region = 0.85

[budgets.ram]
int_sram_shareable      = 0.85
int_sram_no_cacheable   = 0.85
int_dtcm_task_stacks    = 0.90    # leave 10% headroom for stack growth
```

If `int_sram_shareable` ends up at 98% used → emits
`budget.int_sram_shareable` (warning).

---

## `[policies]` — gates and warnings

The largest section. All keys go under one `[policies]` table. Map-typed
fields (`partition_max_bytes`, `forbidden_component_regions`) live in
sub-tables (`[policies.partition_max_bytes]`, etc.).

### Overflow and near-capacity

```toml
[policies]
fail_on_overflow            = true     # default true; flip to false to demote to warning
warn_on_near_capacity_ratio = 0.80     # default 0.90
```

- `overflow.<region>` (**ERROR** if `fail_on_overflow=true`, else WARNING)
  — region utilization exceeds 100%. Always evaluated.
- `near_capacity.<region>` (**WARNING**) — region utilization exceeds
  `warn_on_near_capacity_ratio` but is ≤ 100%. Suppressed for any
  region that has a `[budgets.*]` entry (because that budget warning
  is the more specific signal).

To make warnings fire earlier, drop the threshold:

```toml
[policies]
warn_on_near_capacity_ratio = 0.75    # warn at 75% instead of 90%
```

To disable entirely:

```toml
[policies]
warn_on_near_capacity_ratio = 1.01   # nothing can exceed 101% if not overflowed
```

### Large BSS symbol warnings

```toml
[policies]
warn_on_large_bss_symbol_kb = 4    # default 8
```

For every `.bss` (zero-init) symbol whose size exceeds the threshold,
MemScope emits `large_bss_symbol.<symbol_name>` (WARNING). Useful for
spotting accidentally-large globals like:

```c
static uint8_t debug_buffer[16384];   // 16 KB BSS — would warn at 4 KB threshold
```

Set to `null` (or omit) to disable. The 8 KB default catches common
cases without flooding the report on RTOS apps.

### Hard byte caps with `partition_max_bytes`

Different from `[budgets]`: budgets are **ratios of the region's actual
size**. `partition_max_bytes` are **absolute byte ceilings**, regardless
of how big the linker made the region. Useful when you want to enforce
"the application footprint must never exceed N bytes" even if the
linker layout grows.

```toml
[policies.partition_max_bytes]
APP_FLASH        = 262144       # 256 KB hard cap for application
BOOTLOADER_FLASH = 65536        # 64 KB hard cap for bootloader
```

**Issues emitted:**

- `partition_fit.<region>` (**ERROR**) — region's used bytes exceed the cap
- `partition_near_capacity.<region>` (**WARNING**) — used bytes ≥
  `ota_warn_ratio × cap` (yes, the same ratio is reused for partition
  warnings — defaults to 0.90)
- `partition_fit.unknown_region.<name>` (**WARNING**) — cap references
  a region that doesn't exist (typo detection, mirrors
  `unknown_budget_region`)

### OTA fit gates

For dual-bank firmware with A/B over-the-air updates. Declare which
linker region(s) make up the OTA source bank, plus the per-slot byte
limit that the bootloader enforces.

```toml
[policies]
ota_source_regions   = ["APP_FLASH"]
ota_slot_a_max_bytes = 262144       # 256 KB
ota_slot_b_max_bytes = 262144
ota_warn_ratio       = 0.85         # warn at 85% of slot capacity
```

**Issues emitted:**

- `ota_fit.slot_a` (**ERROR**) — sum of bytes in `ota_source_regions`
  exceeds `ota_slot_a_max_bytes` → wouldn't fit in slot A
- `ota_fit.slot_b` (**ERROR**) — same, for slot B
- `ota_near_capacity.slot_a/slot_b` (**WARNING**) — at or above
  `ota_warn_ratio × slot_max_bytes`
- `ota_source_region.<name>` (**WARNING**) — `ota_source_regions`
  references a region that doesn't exist (typo detection)

Skip the entire OTA block if you don't have A/B firmware — leaving the
fields unset means no OTA gates fire.

### Placement gates: DMA / cacheable / non-cacheable / retention

Four mirrored gates, same shape, different semantics. They all answer
the question "did sections matching pattern X land in region Y, or did
they accidentally drift somewhere they shouldn't?"

```toml
[policies]
# DMA — sections that DMA hardware writes to must be in DMA-safe regions
dma_section_patterns = [".dma_*", ".eth_buffers", ".can_buffers"]
dma_safe_regions     = ["DMA_RAM", "SRAM_NO_CACHEABLE"]

# Non-cacheable — sections shared with peripherals/DMA must skip the cache
non_cacheable_section_patterns = [".nocache_*", ".mcal_bss_no_cacheable"]
non_cacheable_regions          = ["SRAM_NO_CACHEABLE"]

# Cacheable — performance-critical hot paths must stay in cacheable regions
cacheable_section_patterns = [".cache_text", ".hotpath_*"]
cacheable_regions          = ["SRAM_CACHEABLE", "DTCM"]

# Retention — sections that must survive low-power retention sleep
retention_section_patterns = [".retain_*"]
retention_regions          = ["RET_RAM"]
```

**Pattern syntax.** Glob-style on the section name (`*` = any,
`?` = single char). Patterns are case-sensitive.

**Issues emitted** (all **ERROR**):
- `memory_policy.dma.<section>.<region>` — `.dma_*` section in a
  region that's NOT in `dma_safe_regions`
- `memory_policy.non_cacheable.<section>.<region>` — analogous for
  non-cacheable
- `memory_policy.cacheable.<section>.<region>` — analogous for cacheable
- `memory_policy.retention.<section>.<region>` — analogous for retention

If you don't define both halves of a pair (patterns + regions), the
gate is silently inactive — no warnings, no errors, just no checks.

### Component placement: `forbidden_component_regions`

Forbid a logical component from landing in specific regions. **Requires
`[groups]` to be defined** — the matcher uses the group name on the
left.

```toml
[groups]
Bootloader = ["**/boot/**", "**/bootloader/**"]
MCAL       = ["**/Mcal*.o"]

[policies.forbidden_component_regions]
Bootloader = ["SRAM", "DTCM"]      # bootloader code must live in flash
MCAL       = ["APP_FLASH"]          # MCAL must NOT land in app flash
```

**Issue emitted** (**ERROR**):
- `forbidden_component_placement.<group>.<region>` for each violating
  symbol's component+region combo

If the group has no symbols matching, no rule fires. If the group is
not defined in `[groups]`, the rule is silently ignored.

### Build-to-build regression gates

Only effective when run with `--baseline-json <path>` to compare against
a previous report.

```toml
[policies]
max_ram_growth_bytes   = 1024     # warn (or fail with --strict) on >1 KB RAM growth
max_flash_growth_bytes = 4096     # warn on >4 KB flash growth
```

**Issues emitted:**
- `regression.ram_growth` — total RAM grew by more than `max_ram_growth_bytes`
- `regression.flash_growth` — total flash grew by more than `max_flash_growth_bytes`

Severity is **WARNING** by default; **ERROR** when run with `--strict`.
Set to `null` (or omit) to disable.

```bash
memscope --config memscope.toml --strict analyze \
  --elf build/app.elf \
  --baseline-json reports/last-good.json
```

### Suppression

Three knobs, in order of granularity. All take effect AFTER issues are
computed — they hide entries from the Issues section and prevent CI
failures, but the issues still appear in the JSON's `suppressed_issues`
list for traceability.

```toml
[policies]
# Most specific: suppress an exact issue ID
suppress_issue_ids = [
  "large_bss_symbol.tcpip_stacks",        # accepted: TCP stacks are legit large
  "ota_near_capacity.slot_a",
]

# Suppress everything starting with a prefix
suppress_issue_prefixes = [
  "near_capacity.",                       # silence ALL near-capacity warnings
]

# Suppress whole categories (matches the IssueCategory enum)
suppress_categories = [
  "regression",                           # silence build-to-build deltas entirely
]
```

Use the most specific knob you can. `suppress_issue_prefixes` is a
sledgehammer — easy to accidentally hide future issues you actually
want to see.

### Stack and heap detection

No config keys — these are always-on heuristics. MemScope scans the
linker for explicit stack/heap section markers (`.stack`, `_stack_end`,
`.heap`, `_heap_start`, etc.) and emits warnings if missing:

- `missing_stack_definition` (**WARNING**) — no `.stack` section or
  stack symbol found
- `missing_heap_definition` (**WARNING**) — no `.heap` section or heap
  symbol found

Common in some bare-metal apps (no malloc), but worth knowing about.
Suppress with `suppress_issue_ids = ["missing_heap_definition"]` if
your project genuinely doesn't use a heap.

---

## `[groups]` — collapse symbols into logical components

Without `[groups]`, the **Component Ownership** table and **Top
Components by RAM** chart show one row per object file (often hundreds
of rows). With `[groups]`, you collapse those into a handful of named
components.

```toml
[groups]
"MCAL"        = ["**/Mcal*.o", "**/mcal/**", "**/Rte*.o"]
"AUTOSAR_OS"  = ["**/Os_*.o", "**/SafeRTOS*.o"]
"Drivers"     = ["**/drivers/**", "**/Driver*.o"]
"Application" = ["**/app/**", "main.o"]
```

**Pattern matching:**

- Glob-style (`*` matches any chars except `/`, `**` matches any chars
  including `/`, `?` matches single char)
- Case-insensitive
- First match wins (declaration order matters)
- Plain patterns match against multiple ownership fields (`object`,
  `archive`, `compilation_unit`, `source`, `symbol`, `section`)

**Scoped patterns** restrict what to match against — useful when an
object file name overlaps with a source path:

```toml
[groups]
"Vendor_BSP"  = ["object:Std_Types*.o", "object:Compiler*.o"]
"App_Sources" = ["source:**/app/**/*.c"]
"BSS_Heavy"   = ["section:.bss.*"]
```

Available scopes: `object:`, `archive:`, `compilation_unit:` (or
`unit:`), `source:`, `symbol:`, `section:`.

**Important caveat.** If your map file only carries object-file
basenames (no full source paths) — common for IAR/Keil — the `**/path/**`
patterns won't match anything. Switch to bare basename globs:

```toml
[groups]
"MCAL" = ["Adc_*.o", "Aec_*.o", "Mcl_*.o", "Mcu*.o", "Port*.o", "Spi*.o"]
```

If no group matches, MemScope falls back to its built-in heuristic tags
(`Drivers`, `RTOS`, `Middleware`, `Startup`, `Application`), then to
`Unassigned` if even those miss.

---

## `[telemetry]` — opt-in usage data

Off by default. Currently a stub — turning it on with no endpoint set
will validate but transmit nothing.

```toml
[telemetry]
enabled            = false      # master switch
endpoint           = "https://telemetry.example.com/ingest"
emit_command_usage = false
emit_crash_reports = false
```

See [`privacy.md`](privacy.md) for the full data-handling story.

---

## End-to-end recipes

### CI gate: any growth over baseline fails the build

```toml
[policies]
fail_on_overflow                  = true
fail_on_reserved_region_violation = true
warn_on_near_capacity_ratio       = 0.85
max_ram_growth_bytes              = 0     # any RAM growth fails
max_flash_growth_bytes            = 0     # any flash growth fails
```

```bash
memscope --strict --config memscope.toml analyze \
  --elf build/app.elf \
  --baseline-json reports/last-release.json \
  --json reports/current.json \
  --html reports/current.html
```

`--strict` upgrades regression warnings to errors → non-zero exit code → CI fails.

### Bootloader + application split with hard byte budgets

```toml
[policies.partition_max_bytes]
BOOTLOADER_FLASH = 32768        # 32 KB hard cap for bootloader
APP_FLASH        = 458752       # 448 KB for application

[policies]
ota_source_regions   = ["APP_FLASH"]
ota_slot_a_max_bytes = 458752
ota_slot_b_max_bytes = 458752
ota_warn_ratio       = 0.90

[policies.forbidden_component_regions]
Bootloader  = ["SRAM"]          # bootloader code must NOT spill into RAM
Application = ["BOOTLOADER_FLASH"]  # app must NOT pollute bootloader region
```

### DMA-heavy automotive (S32K-style)

```toml
[policies]
dma_section_patterns = [
  ".dma_*",
  ".eth_buffers",
  ".can_buffers",
  ".lin_buffers",
  ".mcal_dma_*",
]
dma_safe_regions = ["int_sram_no_cacheable"]

non_cacheable_section_patterns = [
  ".mcal_bss_no_cacheable",
  ".mcal_data_no_cacheable",
  ".mcal_const_no_cacheable",
]
non_cacheable_regions = ["int_sram_no_cacheable"]
```

Any DMA buffer landing in cacheable RAM → `memory_policy.dma.<section>.<region>` ERROR.

### Noisy report: silence accepted issues

```toml
[policies]
suppress_issue_ids = [
  "missing_heap_definition",                  # bare-metal, no heap by design
  "large_bss_symbol.tcpip_stacks",            # tcpip_stacks legit large
]
suppress_issue_prefixes = [
  "memory_policy.cacheable.",                 # cacheable gate is informational only
]
```

### One TOML, many build variants (single-core / multicore / debug)

Region names that exist in only one variant won't trigger
`unknown_budget_region` (you'd see false positives in CI). The cleanest
pattern is one TOML per variant:

```
configs/
  app-singlecore.toml
  app-multicore.toml
  app-debug.toml
```

Then in CI:

```bash
memscope --config configs/app-singlecore.toml analyze ...
memscope --config configs/app-multicore.toml  analyze ...
```

Each config only mentions regions that actually exist in that variant.
The `unknown_budget_region` warning is your friend here — it tells you
when a config has drifted from its target's linker.

---

## Issue ID reference

Quick lookup of every issue ID prefix MemScope can produce, what
triggers it, and what config knob controls it.

| Issue ID | Severity | Trigger | Config |
|---|---|---|---|
| `overflow.<region>` | ERROR (or WARN) | Region used > 100% | `policies.fail_on_overflow` |
| `budget.<region>` | WARNING | Region used > `[budgets.*]` ratio | `[budgets.flash]` / `[budgets.ram]` |
| `near_capacity.<region>` | WARNING | Region used > `warn_on_near_capacity_ratio` | `policies.warn_on_near_capacity_ratio` |
| `unknown_budget_region.<name>` | WARNING | Budget key references a region that doesn't exist | n/a (typo detector) |
| `large_bss_symbol.<symbol>` | WARNING | BSS symbol size > `warn_on_large_bss_symbol_kb` × 1024 | `policies.warn_on_large_bss_symbol_kb` |
| `partition_fit.<region>` | ERROR | Region used > `partition_max_bytes[<region>]` | `[policies.partition_max_bytes]` |
| `partition_near_capacity.<region>` | WARNING | Region used ≥ `ota_warn_ratio × partition_max_bytes[<region>]` | `[policies.partition_max_bytes]` + `policies.ota_warn_ratio` |
| `partition_fit.unknown_region.<name>` | WARNING | `partition_max_bytes` references a region that doesn't exist | n/a (typo detector) |
| `ota_fit.slot_a` / `ota_fit.slot_b` | ERROR | Sum of `ota_source_regions` bytes > `ota_slot_*_max_bytes` | `policies.ota_*` |
| `ota_near_capacity.slot_a/b` | WARNING | Sum ≥ `ota_warn_ratio × ota_slot_*_max_bytes` | `policies.ota_warn_ratio` |
| `ota_source_region.<name>` | WARNING | `ota_source_regions` references a region that doesn't exist | n/a (typo detector) |
| `regression.ram_growth` | WARNING (ERROR with `--strict`) | RAM grew > `max_ram_growth_bytes` vs baseline | `policies.max_ram_growth_bytes` + `--baseline-json` |
| `regression.flash_growth` | WARNING (ERROR with `--strict`) | Flash grew > `max_flash_growth_bytes` vs baseline | `policies.max_flash_growth_bytes` + `--baseline-json` |
| `forbidden_component_placement.<comp>.<region>` | ERROR | Group's symbols landed in a forbidden region | `[policies.forbidden_component_regions]` + `[groups]` |
| `memory_policy.dma.<section>.<region>` | ERROR | DMA-marked section landed outside `dma_safe_regions` | `dma_section_patterns` + `dma_safe_regions` |
| `memory_policy.non_cacheable.<section>.<region>` | ERROR | Non-cacheable section landed outside `non_cacheable_regions` | `non_cacheable_*` |
| `memory_policy.cacheable.<section>.<region>` | ERROR | Cacheable-marked section landed outside `cacheable_regions` | `cacheable_*` |
| `memory_policy.retention.<section>.<region>` | ERROR | Retention section landed outside `retention_regions` | `retention_*` |
| `missing_stack_definition` | WARNING | No `.stack` section or stack symbol found | n/a (always on; suppress to silence) |
| `missing_heap_definition` | WARNING | No `.heap` section or heap symbol found | n/a (always on; suppress to silence) |
| `placement.ram_section_in_flash..<section>` | WARNING | RAM-init section landed only in flash (no LMA→VMA copy) | n/a (built-in placement check) |
| `placement.unmapped.<section>` | WARNING | Section couldn't be mapped to any region | n/a (built-in placement check) |
| `reserved_region.<region>..<section>` | ERROR | Section landed in a region marked reserved by the linker | `policies.fail_on_reserved_region_violation` |

To suppress any of these, copy the exact ID into
`suppress_issue_ids`, or use a prefix in `suppress_issue_prefixes`
(e.g. `"memory_policy."` to silence all four placement gates at once).
