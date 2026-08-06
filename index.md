# MemScope

**Local-first firmware footprint intelligence for embedded teams.**

MemScope analyzes your ELF, MAP, and linker artifacts to give you a clear picture of where your firmware's flash and RAM are going — and gates your CI build when a regression sneaks in. Built for embedded engineers who want SonarQube-style enforcement without sending build artifacts to a cloud.

```bash
pip install memscope-fw
```

<!--
README IMAGE HOSTING

The MemScope source repo is private and PyPI renders this README as
standalone HTML, so the screenshots below have to come from a public
URL. They are mirrored in `adrian-dumitrescu/memscope-assets` (a tiny
public repo that holds only the PNGs — no source code).

The canonical source for these PNGs is `docs/assets/screenshots/` in
THIS repo. The mirror runs automatically on every push to `main` that
touches `docs/assets/screenshots/**` — see
`.github/workflows/publish-assets.yml` (parallel to the existing
publish-docs.yml that mirrors docs/public/** to memscope-docs).

To refresh: edit the demo fixtures (`docs/demo/demo.map` etc.), re-run
the analyze + Playwright capture flow to regenerate the PNGs in
`docs/assets/screenshots/`, then commit + push to main. The workflow
takes care of the public repo.

If the assets-repo arrangement is ever retired (source repo goes
public, or a memscope.dev landing page takes over hosting), swap the
URL prefix below in one sed pass and delete the workflow. The local
PNGs stay; only the URL prefix changes.
-->

## Screenshots

A single `memscope analyze` run on a representative STM32H7-class firmware (Cortex-M7 + FreeRTOS + lwIP + USB CDC, 6 memory regions, ~100 contributing input sections, ~150 symbols) produces this self-contained interactive HTML report — open it in any browser, no internet required, no JS framework to install. Headline cards surface the **tightest region** (here, RAM_D1 at 99.8% capacity — the kind of near-overflow that's invisible until you flash silicon), total Flash / RAM consumption, and the alignment-waste budget you could reclaim:

![MemScope analyze HTML report — headline view](https://raw.githubusercontent.com/adrian-dumitrescu/memscope-assets/main/screenshots/analyze-hero.png)

Below the fold: **Memory Layout Explorer** (interactive section bars per region), **Visual Insights** (top contributors / functions / globals), **Region Utilization** (per-region capacity, alignment overhead, fragmentation), **Section Explorer** (drill-down with byte-level lookup), **Component Ownership**, **Issues and Recommendations**, **Comparison vs baseline**, and **Diagnostics and Assumptions** so reviewers can see exactly what was inferred vs measured. [See the full-page screenshot →](https://raw.githubusercontent.com/adrian-dumitrescu/memscope-assets/main/screenshots/analyze-html-report.png)

`memscope diff` compares two builds and highlights the regressions that matter — headline byte deltas, per-region growth chart, top regressions and reductions tables, plus a deterministic JSON contract your CI can gate on:

![MemScope diff report — build-to-build comparison](https://raw.githubusercontent.com/adrian-dumitrescu/memscope-assets/main/screenshots/diff-html-report.png)

The fixtures used to generate these screenshots are in [`docs/demo/`](docs/demo/) — reproducible with `python -m memscope analyze --map docs/demo/demo.map --linker docs/demo/demo.ld --html report.html`.

> **License at a glance.** MemScope is **free** — the `memscope-fw` distribution on PyPI is gratis for any use, including commercial use by companies of any size. No license key, no watermark, no time limit, no seat count, no procurement required, and no feature locked behind a purchase. Run `analyze`, `validate`, `diff`, and `--strict` CI gating today, indefinitely, on as many machines and build agents as you like. Need something MemScope doesn't ship — a toolchain it can't parse, an internal format, a bespoke adaptation? That's arranged separately by email; it isn't part of this distribution and doesn't change your right to keep using it. Closed-source proprietary licence — not OSI / FSF open source. Full text: see the **License** section on this PyPI page (or the `LICENSE` file shipped inside the installed wheel).

---

## What it does

- **Inspects** ELF + MAP + linker scripts and reconstructs the per-region, per-section, per-symbol, per-component memory layout — everything that's actually placed in your firmware image and what each part costs.
- **Reports** in three forms from a single run: a compact terminal summary for CI logs, a versioned JSON for scripts and dashboards, and a self-contained interactive HTML for engineers to explore.
- **Gates** CI builds when a budget is busted, a region overflows, or a baseline diff exceeds your threshold — fail-fast with deterministic exit codes.
- **Compares** two builds side-by-side via `memscope diff`, showing top regressions and reductions across regions, sections, symbols, and components.
- **Runs entirely offline.** MemScope makes no network calls. No build artifacts ever leave your machine. No cloud backend.

## Why teams pick MemScope over rolling their own scripts

| Capability | Hand-rolled scripts | MemScope |
|---|---|---|
| Parse ELF + MAP + linker config consistently across toolchains | Different per project | One CLI, GNU + IAR + Keil + Clang/LLD all supported (dedicated parsers per toolchain, normalized to a single internal model) |
| Stable JSON schema your CI can rely on | Brittle | Versioned + `validate` subcommand |
| Drop-in CMake integration | DIY | `cmake/FirmwareFootprint.cmake` |
| Interactive HTML report for design reviews | None | Self-contained, no internet needed to open |
| CI gating with friendly error messages | grep + bash | `--strict` exit codes |
| Multi-core / multi-region partition awareness | Usually missing | First-class |

## Install

```bash
pip install memscope-fw
```

Requires Python ≥ 3.10. Wheels are published per-platform for Windows (x86_64) + Linux (manylinux2014, x86_64) + macOS (Apple Silicon / arm64), each on Python 3.10 / 3.11 / 3.12 / 3.13 / 3.14.

## 5-minute quick start

```bash
# 1. Analyze a single build — produces report.html you can open in any browser
memscope analyze \
  --elf path/to/firmware.elf \
  --map path/to/firmware.map \
  --linker path/to/linker.ld \
  --json report.json \
  --html report.html

# 2. Compare two builds (e.g. before/after a refactor)
memscope diff \
  --current-json report.json \
  --baseline-json baseline.json \
  --html diff.html

# 3. Gate CI on a regression budget (exits non-zero if violated)
memscope analyze --strict \
  --elf firmware.elf --map firmware.map --linker linker.ld \
  --baseline-json baseline.json
```

The HTML report is **fully self-contained** — D3, ECharts, and all interactive visualizations are bundled inline, so it works in air-gapped CI environments and can be archived as a single file.

## What you get

Everything below is in the `memscope-fw` distribution on PyPI, free for
any use including commercial use inside a company. There is no paid
tier, no feature gate, and no upgrade to buy.

- `analyze` — full ELF/MAP/linker analysis, all reports
- `validate` — JSON-schema sanity check, ideal as a CI-pipeline wedge
- `diff` — baseline comparison + interactive visualization
- `--strict` exit codes for CI gating
- Custom `[budgets]` / `[policies]` config sections, including issue suppression
- CSV export
- Plain-text + markdown memory-layout dumps (`--text` / `--markdown`) — full-memory hierarchical region → section → top-symbols trees, designed for PR comments, build logs, and `diff`-friendly snapshots
- HTML report with full Memory Layout Explorer
- Multi-core / partitioned-SoC analysis
- All current and future toolchain parsers (GNU, IAR, Keil, Clang/LLD)

### Need something MemScope doesn't ship?

Custom extensions and bespoke engineering — a parser for a toolchain or
proprietary linker format that isn't supported, an adapter for an
internal build format, an adaptation to your environment — are
available by arrangement. That work is separate software under its own
written agreement; it is not part of this distribution, is not
published on PyPI, and nothing about it changes your right to keep
using MemScope under the licence above.

Email <dumitrescu.adrian121@gmail.com> and describe what you need.

## Documentation

Public documentation (CLI reference, CMake integration, CI tutorial, report interpretation guide, config schema, troubleshooting, privacy & GDPR posture, enterprise procurement Q&A) ships in [`docs/public/`](docs/public/) and is also published at <https://adrian-dumitrescu.github.io/memscope-docs/>.

A condensed walkthrough is available via:

```bash
memscope --help                         # top-level commands
memscope analyze --help                 # per-command reference
```

Every subcommand has built-in `--help` documentation.

## Privacy posture

- MemScope makes **no network calls of any kind** in normal operation. Your ELF / MAP files never leave your machine.
- There is **no licensing backend, no activation, no licence check, and no sub-processors**. Nothing about your use is reported to anyone.
- Opt-in telemetry is OFF by default; if enabled in `memscope.toml`, the User specifies the recipient endpoint — the Licensor neither receives nor stores telemetry.
- Full privacy policy: [`docs/public/privacy.md`](docs/public/privacy.md).

## Support

For bug reports, integration help, questions about the licence, and custom engineering enquiries:

- **Email**: <dumitrescu.adrian121@gmail.com>

When reporting an issue, please include the output of `memscope diagnostics export` — it bundles version, environment, and parser diagnostics into a single ZIP that's safe to attach.

## License

Closed-source proprietary EULA. The binding text — with a Plain-English summary at the top — is in the `LICENSE` file shipped inside the wheel, and surfaced in the **License** section on the PyPI project page.

In short:

- **MemScope (the PyPI distribution) is gratis for any use, including commercial use inside a company.** No license key, no seat count, no time limit, no payment, no contact with the maintainer required, and no feature locked behind a purchase.
- **Custom extensions and bespoke engineering** are arranged separately (clause 3). Any such work is separate software under its own written agreement, is not published on PyPI, and is not licensed under this LICENSE. Enquiries: <dumitrescu.adrian121@gmail.com>.
- **Not open source.** No rights are granted under any OSI-approved or FSF-recognised license. Source code is not distributed; you receive the binary wheel published on PyPI.
