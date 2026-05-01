# MemScope — Enterprise procurement / security review one-pager

> Audience: corporate procurement, security, and legal reviewers
> evaluating whether `memscope-fw` (PyPI) can be installed and used
> inside your organisation. This page answers the questions that
> typically come up in vendor security questionnaires.
>
> **One-line summary:** The MemScope core (`memscope-fw` on PyPI) is
> licensed gratis for any use, including commercial use by companies
> of any size, runs entirely locally with no network dependency, and
> can be installed today. Optional commercial Bundles are delivered
> separately as Customer-Bound Wheels under purchase contracts and
> are not yet generally available for sale.

---

## 1. Licensing

| Question | Answer |
|---|---|
| What license? | Closed-source proprietary EULA. License expression: `LicenseRef-MemScope-EULA`. Full text: included in the wheel METADATA (PEP 639) and at the project page on PyPI. |
| Is the source code published? | **No.** Source code is not distributed. You receive the binary wheel published on PyPI. |
| Can a company use the core for commercial work? | **Yes.** Clauses 2 of the LICENSE explicitly grant core-Software use to "individuals, contractors, public bodies, and companies (including for-profit corporations) for internal business purposes, including in commercial product development workflows", without payment, indefinitely. |
| What's available in the gratis PyPI distribution? | Everything currently shipped: `analyze`, `validate`, `diff`, all reports (terminal / JSON / HTML / CSV), `--strict` CI gating, custom `[budgets]` / `[policies]` / `[suppressions]` config sections, all current toolchain parsers (GNU, Keil, Clang/LLD), Memory Layout Explorer, all visualizations. |
| What's reserved for paid Bundles? | Optional commercial extensions: PR/Slack/email notifications integrations, SoC-family analysis extras, ISO 26262 / IEC 62304 / DO-178C compliance evidence Bundles, custom toolchain support (IAR, Renesas, Microchip, etc.) for users that need it, bespoke per-customer features. **None of these are needed for the free core to work end-to-end.** |
| Is the free core time-limited? | **No.** No trial countdown, no expiry, no devolution. The PyPI distribution runs forever, identically, on day 1 and day 10000. |
| Are paid Bundles available now? | **Not yet.** Bundle licences are not on sale; the Licensor's commercial entity is being registered. The free core is fully operational today. |
| Governing law / jurisdiction? | Romanian law; courts of Bucharest, Romania (LICENSE clause 11). EU consumer-protection rights are preserved for natural persons. |
| Custom EULA / master agreement? | Available on request once Bundle sales begin. Contact dumitrescu.adrian121@gmail.com. |

> **Why your dependency scanner may flag this:** the license expression
> `LicenseRef-MemScope-EULA` is a custom (non-SPDX) identifier, which
> tools like Black Duck, Snyk, Dependency-Track, FOSSA, and pip-audit
> route through manual review by default — that is the **correct**
> behaviour for proprietary software, not a bug. This page exists so
> your reviewer can resolve the manual-review queue in 5 minutes.

## 2. Privacy and data flow

| Question | Answer |
|---|---|
| Does the free core phone home? | **No, ever.** The `memscope-fw` core distribution makes **zero outbound network calls** in normal operation. It does not contact a licensing backend, telemetry endpoint, update checker, or any other server. |
| What about paid Bundles? | Subscription Bundles (when purchased) contact the licensing backend (Keygen.sh) on first activation and on periodic refresh (default: once per 7 days during a 7-day offline grace window). One-Time Bundles (bespoke or compliance) make NO network calls — identical to the free core. |
| What does the licensing backend see (Subscription Bundles only)? | License key, machine fingerprint hash (SHA-256 of hostname + OS + arch + per-install UUID), OS summary (`platform.uname()` data), Bundle name + version. **No analysis content, no file paths, no symbol names, no source code.** Detail: [`privacy.md`](privacy.md). |
| GDPR posture? | Free core: the Licensor is neither a controller nor a processor (no data flows). Subscription Bundles only: the Licensor is the data controller for the activation payload; Keygen.sh is the sub-processor under a DPA. EU residency available on request. Full posture: [`privacy.md`](privacy.md) GDPR section. |
| Data Processing Addendum (DPA)? | Template skeleton exists internally; final lawyer-reviewed DPA available for enterprise customers entering Bundle contracts. Request via dumitrescu.adrian121@gmail.com. |
| Build artefacts (ELF / MAP / linker scripts) — do they leave the customer's network? | **No, ever.** All analysis runs locally. MemScope does not have an upload endpoint, even opt-in. |
| Telemetry? | **Off by default.** If enabled (`[telemetry]` block in `memscope.toml`), the customer specifies the endpoint URL — the Licensor neither receives nor stores telemetry. The whitelisted payload is documented in [`privacy.md`](privacy.md). |

## 3. Supply-chain security

| Question | Answer |
|---|---|
| Where is the package hosted? | Free core: PyPI <https://pypi.org/project/memscope-fw/>. Bundles: delivered directly by the Licensor under purchase contracts (no public package index). |
| Is the release pipeline reproducible / auditable? | The release workflow is `.github/workflows/release.yml` in the (private) source repo. Wheels are built per-platform on GitHub-hosted runners. |
| Is the package signed? | **PEP 740 attestations** are emitted via PyPI Trusted Publishing (Sigstore-backed). Verify with `pypi-attestations verify pypi --repository <project-repo> <wheel-file>`. |
| Software Bill of Materials (SBOM)? | **CycloneDX SBOMs** (JSON + XML) are attached to each GitHub Release of the free core. Direct deps: `httpx`, `jinja2`, `pydantic`, `pynacl`, `rich`, `typer` — see `pyproject.toml` `[project.dependencies]`. |
| Package integrity check before install? | `pip install` checks PyPI hash + size automatically. For belt-and-suspenders verification: `pip download memscope-fw && pip hash memscope_fw-*.whl` and compare against the PyPI page. |
| Vendored / bundled third-party code? | The HTML report bundles `d3.v7.9.0.min.js` and `echarts.v5.5.1.min.js` (verbatim copies, SHA-256-pinned in `src/memscope/outputs/frontend_runtime.py`). Both are licensed under permissive open-source licenses (BSD-3 for D3, Apache-2.0 for ECharts) compatible with closed-source distribution. |

## 4. Operational dependencies

| Question | Answer |
|---|---|
| Does MemScope require an internet connection at runtime? | **Free core: no, ever.** Subscription Bundles: only for activation and validation. Up to 7 days of offline operation between validations. Suitable for air-gapped CI (automotive, defence, industrial). One-Time Bundles: never. |
| Python version support? | 3.11, 3.12, 3.13, 3.14. |
| Operating system support? | Windows (x86_64), Linux (manylinux2014, x86_64), macOS (Apple Silicon / arm64). Wheels published per-platform per-Python-version. |
| Native code? | Both the free core and Customer-Bound Wheels ship as platform-specific wheels with the algorithmic IP modules (parsers, analyzers, diff, rule evaluators, toolchain inference) Cython-compiled to `.pyd` (Windows) / `.so` (Linux/macOS). Customer-Bound Wheels additionally compile the customer-binding constants and licensing modules. The CLI surface, domain types, and a few utility modules ship as plain `.py`. |
| What gets installed under `~/`? | A small state directory at `~/.memscope/` containing `eula_state.json` (EULA acceptance record) and an `install_id` UUID. Subscription Bundles additionally write `license_cache.json` after activation. Total footprint: < 10 KB. |

## 5. Vulnerability response

| Question | Answer |
|---|---|
| How do I report a security issue? | Email dumitrescu.adrian121@gmail.com with subject `SECURITY:` and a clear description. We aim to acknowledge within 3 business days and have a fix or mitigation timeline within 14 days. |
| Coordinated disclosure? | Yes — please give us a 90-day disclosure window for high-severity issues. We will publish a fix, advisory, and credit you (if you wish) at the end of the window or sooner if a fix is shipped. |
| CVE issuance? | We will request CVEs for confirmed vulnerabilities affecting shipped releases. |

## 6. Standard procurement-questionnaire shortcuts

If you have a templated security questionnaire to send, the following
boilerplate answers are correct for current MemScope:

- **SOC 2 / ISO 27001:** not yet certified (small commercial entity not yet registered). The licensing backend used by Subscription Bundles (Keygen.sh) is SOC 2 Type II.
- **Pen test cadence:** internal; external pen test of the licensing backend planned alongside Bundle launch.
- **Customer data isolation:** N/A for the free core (processes no customer data on the Licensor's infrastructure). Subscription Bundle activations are isolated per customer in the Keygen-managed backend.
- **Encryption in transit:** TLS 1.2+ for all licensing-backend traffic (Subscription Bundles only).
- **Encryption at rest:** Keygen handles backend encryption. Local files (`~/.memscope/license_cache.json` for Subscription Bundles, `eula_state.json` for the core) are HMAC-signed and stored at OS-default file permissions; sensitive material does not travel between machines.
- **Right-to-audit:** subject to terms in the DPA (request via support email).
- **Sub-processors:** Keygen, Inc. (US, with EU residency on request) — used only by Subscription Bundles. The free core uses no sub-processors.

---

## Contact

- **Licensing, commercial, and contractual:** dumitrescu.adrian121@gmail.com
- **Security:** same address, prefix subject with `SECURITY:`
- **PyPI project page:** <https://pypi.org/project/memscope-fw/>
- **Documentation:** see the `docs/public/` directory bundled with the
  source distribution, or the rendered version at the project's
  documentation URL.

This page is updated whenever the licensing posture, supply-chain
configuration, or sub-processor list changes. Last updated: 2026-05-01.
