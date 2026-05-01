# MemScope Report Interpretation Guide

MemScope reports mix direct measurements from artifacts with inferred model information. This guide helps interpret both correctly.

## Where values come from

## Measured values (high confidence)

Usually direct from ELF/MAP/linker records:

- section sizes (`sections[*].size`)
- section addresses (`vma`, `lma`)
- symbol sizes and addresses
- region definitions from linker `MEMORY` blocks

## Inferred values (model-dependent)

Computed or inferred from multiple sources:

- section execution/load-region mapping when one source is missing
- stack/heap hints and notes
- component ownership grouping
- initialized RAM flash-init payload estimate
- padding/alignment waste estimate
- policy outcomes based on configured thresholds

## JSON report reading order

1. `metadata`: tool version, schema version, target, detected toolchain.
2. `summary`: total flash/RAM and region utilization.
3. `issues`: actionable policy or overflow/reserved placement issues.
4. `sections` and `regions`: raw footprint layout.
5. `symbols_summary` and `components`: top growth contributors.
6. `diff`: baseline regression/reduction status and ranking.
7. `diagnostics`: assumptions, unresolved ambiguities, policy trace, license/telemetry context.

## Diagnostics fields that matter most

- `diagnostics.parse_assumptions`: what MemScope had to assume (for example missing ELF).
- `diagnostics.unresolved_ambiguities`: unresolved parse/model uncertainty.
- `diagnostics.policy`: which rules fired and which issue IDs were produced.
- `diagnostics.stack_heap_hints`: evidence used for stack/heap interpretation.
- `diagnostics.multicore` (when present): inferred per-core partitioning of regions/sections/symbols.
- `memory_policy.*` issues: advanced DMA/cacheability/retention placement violations.

Interpretation rule:

- if diagnostics show unresolved ambiguity, treat derived metrics as directional and verify with linker/map context.

## HTML report usage notes

- Region bars cap visually at `100%`, but numeric values may exceed `100%` on overflow.
- Tables are sortable and intended for quick triage of biggest contributors.
- `DEVELOPER MODE` watermark means explicit developer mode was used.

### What the table columns mean

The **Top Functions** and **Top Globals** tables share the same five columns:

| Column | Meaning |
|---|---|
| **Name** | Symbol name from the ELF symbol table. C++ names are demangled where possible. |
| **Region** | The memory region the symbol's bytes live in (e.g. `int_pflash_region`, `int_sram_region`). Same address range that shows in the Memory Layout map. |
| **Component** | A logical grouping defined by you in `[groups]` in `memscope.toml` (e.g. `MCAL`, `Application`, `Drivers`). When you don't define `[groups]`, this falls back to the object-file name — that's why **Component and Object can look identical**. Set `[groups]` to collapse hundreds of `.o` rows into a handful of meaningful groups. |
| **Object** | The literal compilation unit (`.o` file) the symbol came from. Always present. |
| **Size (B)** | **For Top Functions**: the size in bytes of the function's compiled machine code (instructions) that the linker placed in flash. **For Top Globals**: the size in bytes the variable occupies at link time — `.data` payload, `.bss` reservation, or `.rodata` constant. **Never includes**: stack usage, heap allocations, or local-variable footprint. |

Hover any table column header in the rendered HTML to see the same information as a tooltip.

> **Stack usage is a runtime concept.** It's NOT in any of these tables — it requires call-graph analysis or compiler-emitted `.su` files (GCC `-fstack-usage`) which MemScope doesn't currently parse. Local variables don't appear here either; they live on the stack at runtime, not in any linker-managed region.

### Interactive Memory Layout Explorer

The `Memory Layout` section is a single composite canvas that stacks every
memory region in address order. Reading it:

- **Region bands** use the y-axis for address and the x-axis for utilization.
  A band's header shows the region name, total size, used/free bytes, and
  its render mode (e.g. `segmented-stacked`, `scaled-for-readability`).
- **Ranges** inside each band are colored by `role` (code, data, bss, heap,
  stack, reserved, protected, free) and stroke-highlighted when they
  participate in a policy issue.
- **Zoom/pan**: mousewheel or pinch zooms up to `EXPLORER_MAX_ZOOM`
  (currently 10 000 x); drag to pan. Arrow keys step the focused range,
  `Enter` opens the side panel.
- **Focus & history**: the region select and prev/next buttons walk the
  zoom history without losing the current zoom position.
- **Minimap**: the strip under the canvas is a live minimap — click a
  region to jump to it; the viewport rectangle tracks the current zoom.

### Analytical lenses

Three toggles above the canvas recolor the composite without rebuilding it
(zoom/pan state is preserved):

- **Alignment Lens** — recolors padding ranges by their power-of-two
  alignment class. Use it to spot alignment-heavy waste (e.g. a sea of
  64-byte padding suggests cache-line-aligned structs).
- **Padding Heatmap** — overlays a per-region padding-density heat strip.
  Use it to find regions that are syntactically "full" but actually hold
  mostly padding.
- **Memory Roles** — recolors sections and adds role badges
  (`BSS` / `DATA` / `CODE` / `HEAP` / `STACK` / `VECTORS`). Use it to
  validate that your linker script placed roles where you expect them.

### Cross-filter

The filter controls on the tabular sections (region, component, section
name, issue text, issue severity) share state with the explorer:

- Selecting a region in a table dims every non-matching range in the
  composite and the minimap rather than re-rendering; zoom/pan survive.
- Selecting a range in the composite sets the matching filters on the
  tables (single source of truth is the host-page `sharedFilters` object).
- The explorer's `Reset` control clears filters globally.

### Command palette, address inspector, share menu

- **Command palette** (`Ctrl K`) jumps to any section, region, or shipped
  command (toggle theme, reset filters, open share menu, etc.).
- **Address inspector** (`Address` button) accepts hex or decimal and
  resolves to the owning region / section / symbol when input data allows.
- **Share menu** (`Share` button) provides:
  - *Copy link to current view* — a shareable URL that encodes the current
    filter/zoom state (offline-safe; it is a `file://` or host URL fragment).
  - *Copy report summary as Markdown* — paste-ready summary block for PR
    descriptions and tickets.
  - *Download memory layout (SVG / PNG)* — exports the composite canvas
    as a static asset for slide decks or bug reports.

### Accessibility and keyboard control

- The app-bar, side rail, tabs, and command palette are keyboard-navigable.
- The composite canvas is focusable (`tabindex=0`). Arrow keys step ranges,
  `Enter` opens the side panel, `Escape` closes pop-ups.
- All hex addresses and byte counts use `font-variant-numeric: tabular-nums`
  for clean column alignment.

## Diff interpretation

- Positive delta: growth/regression.
- Negative delta: reduction.
- `diff.status != "ok"` means comparison was not valid or not requested.

Focus triage on:

1. blocking issues (`severity=error`)
2. top regressions by bytes
3. regions near/over policy thresholds
