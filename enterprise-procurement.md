# MemScope — Enterprise procurement / security review one-pager

> Audience: corporate procurement, security, and legal reviewers
> evaluating whether `memscope-fw` (PyPI) can be installed and used
> inside your organisation. This page answers the questions that
> typically come up in vendor security questionnaires.
>
> **One-line summary:** MemScope (`memscope-fw` on PyPI) is licensed
> gratis for any use, including commercial use by companies of any
> size, runs entirely locally with **no network dependency and no
> licensing backend**, and can be installed today. There is no paid
> tier, no licence key, and no feature gated behind a purchase.

---

## 1. Licensing

| Question | Answer |
|---|---|
| What license? | Closed-source proprietary licence. License expression: `LicenseRef-MemScope-EULA`. Full text: included in the wheel METADATA (PEP 639) and at the project page on PyPI. |
| Is the source code published? | **No.** Source code is not distributed. You receive the binary wheel published on PyPI. |
| Can a company use it for commercial work? | **Yes.** Clause 2(b) of the LICENSE grants use "for any purpose, INCLUDING use by a company, organisation, or public body in connection with its commercial activities, and including use in automated build and continuous-integration systems", on "any number of machines the User owns or is authorised to operate", without payment, indefinitely. |
| What's in the distribution? | Everything MemScope has: `analyze`, `validate`, `diff`, all reports (terminal / JSON / HTML / CSV), `--strict` CI gating, `[budgets]` / `[policies]` config including issue suppression, multi-core / partitioned-SoC analysis, all toolchain parsers (GNU, IAR, Keil, Clang/LLD), Memory Layout Explorer, all visualizations. |
| Is anything reserved for a paid tier? | **No.** There is no paid tier and no feature gate. Every command and every output format in the product is in the distribution you install. |
| Is it time-limited? | **No.** No trial countdown, no expiry, no devolution, no activation. It runs identically on day 1 and day 10000. |
| Is there a licence key or seat count? | **No.** Nothing to purchase, register, activate, or renew. Install on as many workstations and build agents as you like. On first run it displays a summary of the terms and asks you to accept once; that happens locally and transmits nothing. In CI, `memscope accept-terms` records acceptance non-interactively. |
| What about work MemScope doesn't support? | Custom extensions and bespoke engineering (e.g. a parser for a toolchain or proprietary linker format not currently supported, an adapter for an internal build format) are arranged separately under clause 3. Any such work is **separate software under its own written agreement**, is not distributed on PyPI, and is not licensed under this LICENSE. It does not affect your licence to MemScope. Enquiries: dumitrescu.adrian121@gmail.com. |
| Governing law / jurisdiction? | Romanian law; courts of Bucharest, Romania (LICENSE clause 10). EU consumer-protection rights are preserved for natural persons. |
| Custom master agreement? | The Licensor's then-current agreement and a Data Processing Addendum are available on request for organisations entering significant commercial relationships. Contact dumitrescu.adrian121@gmail.com. |

> **Why your dependency scanner may flag this:** the license expression
> `LicenseRef-MemScope-EULA` is a custom (non-SPDX) identifier, which
> tools like Black Duck, Snyk, Dependency-Track, FOSSA, and pip-audit
> route through manual review by default — that is the **correct**
> behaviour for proprietary software, not a bug. This page exists so
> your reviewer can resolve the manual-review queue in 5 minutes.

## 2. Privacy and data flow

| Question | Answer |
|---|---|
| Does it phone home? | **No, ever.** MemScope makes **zero outbound network calls** in normal operation. It does not contact a licensing backend, telemetry endpoint, update checker, or any other server. |
| Is there a licensing backend? | **No.** There is no activation, no licence check, and no licensing service of any kind. The distribution contains no licensing client — the release pipeline asserts their absence on every published wheel (`scripts/wheel_smoke_test.py`). |
| GDPR posture? | The Licensor is **neither a data controller nor a data processor** with respect to your use of MemScope. No personal data is collected, transmitted, or stored by the Licensor. Full posture: [`privacy.md`](privacy.md). |
| Sub-processors? | **None.** MemScope uses no sub-processors, no analytics provider, and no third party receiving any data about your use of the software. |
| International data transfers? | **None.** No data leaves your machine, so no transfer mechanism (SCCs, adequacy decision) is engaged. |
| Data Processing Addendum (DPA)? | Because the Licensor performs no processing for MemScope, a DPA has no subject matter to cover. One can still be provided on request for organisations whose procurement process requires the artefact. Request via dumitrescu.adrian121@gmail.com. |
| Build artefacts (ELF / MAP / linker scripts) — do they leave the customer's network? | **No, ever.** All analysis runs locally. MemScope has no upload endpoint, not even opt-in. |
| Telemetry? | **Off by default.** If you enable it (`[telemetry]` block in `memscope.toml`), **you** specify the endpoint URL and **you** are the data controller for what reaches it — the Licensor neither receives nor stores telemetry. The payload is exactly `version`, `command`, `runtime_ms`; a whitelist in code drops anything else. Detail: [`privacy.md`](privacy.md). |
| Are reports watermarked or licence-stamped? | **No.** Reports carry no licence identifiers, no watermarks, and no attestation footer. |

## 3. Supply-chain security

| Question | Answer |
|---|---|
| Where is the package hosted? | PyPI <https://pypi.org/project/memscope-fw/>. |
| Is the release pipeline reproducible / auditable? | The release workflow is `.github/workflows/release.yml` in the (private) source repo. Wheels are built per-platform on GitHub-hosted runners. |
| Is the package signed? | **PEP 740 attestations** are emitted via PyPI Trusted Publishing (Sigstore-backed). Verify with `pypi-attestations verify pypi --repository <project-repo> <wheel-file>`. |
| Software Bill of Materials (SBOM)? | **CycloneDX SBOMs** (JSON + XML) are attached to each GitHub Release. Direct runtime deps: `httpx`, `jinja2`, `pydantic`, `rich`, `typer`, plus `tomli` on Python 3.10 only — see `pyproject.toml` `[project.dependencies]`. |
| Why does it depend on an HTTP client if it makes no network calls? | `httpx` is the transport for opt-in telemetry only. With telemetry off — the default — it is imported but never used to issue a request. |
| Package integrity check before install? | `pip install` checks PyPI hash + size automatically. For belt-and-suspenders verification: `pip download memscope-fw && pip hash memscope_fw-*.whl` and compare against the PyPI page. |
| Vendored / bundled third-party code? | The HTML report bundles `d3.v7.9.0.min.js` and `echarts.v5.5.1.min.js` (verbatim copies, SHA-256-pinned in `src/memscope/outputs/frontend_runtime.py`). Both are licensed under permissive open-source licenses (BSD-3 for D3, Apache-2.0 for ECharts) compatible with closed-source distribution. |

## 4. Operational dependencies

| Question | Answer |
|---|---|
| Does MemScope require an internet connection at runtime? | **No, ever.** Suitable for air-gapped CI (automotive, defence, industrial) with no special configuration — there is no activation to perform and nothing that expires. |
| Python version support? | 3.10, 3.11, 3.12, 3.13, 3.14. |
| Operating system support? | Windows (x86_64), Linux (manylinux2014, x86_64), macOS (Apple Silicon / arm64). Wheels published per-platform per-Python-version. |
| Native code? | Wheels are platform-specific, with the algorithmic IP modules (parsers, analyzers, diff, rule evaluators, toolchain inference) Cython-compiled to `.pyd` (Windows) / `.so` (Linux/macOS). The CLI surface, domain types, and a few utility modules ship as plain `.py`. |
| What gets installed under `~/`? | A small state directory at `~/.memscope/` containing `eula_state.json` (terms-acceptance record) and an `install_id` UUID that is never transmitted. Total footprint: < 10 KB. Deleting the directory resets MemScope to its never-installed state. |

## 5. Vulnerability response

| Question | Answer |
|---|---|
| How do I report a security issue? | Email dumitrescu.adrian121@gmail.com with subject `SECURITY:` and a clear description. We aim to acknowledge within 3 business days and have a fix or mitigation timeline within 14 days. |
| Coordinated disclosure? | Yes — please give us a 90-day disclosure window for high-severity issues. We will publish a fix, advisory, and credit you (if you wish) at the end of the window or sooner if a fix is shipped. |
| CVE issuance? | We will request CVEs for confirmed vulnerabilities affecting shipped releases. |

## 6. Standard procurement-questionnaire shortcuts

If you have a templated security questionnaire to send, the following
boilerplate answers are correct for current MemScope:

- **SOC 2 / ISO 27001:** not certified. The Licensor operates no service
  in connection with MemScope — there is no backend, no hosted
  component, and no customer data on Licensor infrastructure — so these
  certifications have no in-scope system to cover.
- **Pen test cadence:** not applicable; no network-reachable service exists.
- **Customer data isolation:** N/A — MemScope processes no customer data
  on the Licensor's infrastructure. All processing is on your machines.
- **Encryption in transit:** N/A — no data is transmitted. If you enable
  telemetry, the transport is HTTPS to the endpoint you configure.
- **Encryption at rest:** N/A on the Licensor side. Local files under
  `~/.memscope/` are stored at OS-default file permissions and contain
  no personal data beyond what your own machine already holds.
- **Right-to-audit:** no Licensor-side processing exists to audit. You
  can audit the software itself — see [`trust-model.md`](trust-model.md).
- **Sub-processors:** none.
- **Business continuity / vendor lock-in:** MemScope needs no server, so
  there is no service that can go away. An installed wheel keeps working
  indefinitely regardless of the Licensor's status. Its outputs (JSON,
  CSV, HTML) are open formats with a versioned, documented JSON schema.

---

## Contact

- **Licensing, commercial, and contractual:** dumitrescu.adrian121@gmail.com
- **Security:** same address, prefix subject with `SECURITY:`
- **PyPI project page:** <https://pypi.org/project/memscope-fw/>
- **Documentation:** see the `docs/public/` directory bundled with the
  source distribution, or the rendered version at the project's
  documentation URL.

This page is updated whenever the licensing posture or supply-chain
configuration changes. Last updated: 2026-08-06 (LICENSE Version 3).
