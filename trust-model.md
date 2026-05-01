# MemScope Trust Model

## Local-Only By Default

- Build artifact analysis is local-only.
- Remote/network artifact paths are rejected.
- No binary upload path exists in the analysis pipeline.

## Telemetry and Monitoring

- Telemetry is disabled by default.
- Telemetry configuration is explicit opt-in and currently implemented as a
  future-safe no-op stub in v1.
- Monitoring focus is limited to licensing/support workflows.

## Network Interaction Boundaries

- Normal analysis commands (`analyze`, `diff`, `validate`, `diagnostics export`)
  rely on local artifacts and local license cache checks.
- Online license verification is only attempted during explicit licensing
  actions (for example `license status` and activation/deactivation flows).

## Privacy Guarantees

- Reports are generated locally as JSON/HTML/CSV files.
- Diagnostics bundle export sanitizes filesystem paths and redacts sensitive
  metadata fields (tokens, keys, fingerprints, signatures).

## Offline Operation

- Analysis works fully offline.
- After activation, cached license state supports offline commercial use
  according to license state policy.
