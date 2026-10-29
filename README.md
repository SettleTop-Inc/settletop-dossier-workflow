# SettleTop AI Dossier — private-door workflow

Public, workflow-only repo. It holds one reusable GitHub Actions workflow —
[`.github/workflows/private-door.yml`](.github/workflows/private-door.yml) — that a
company runs **inside its own GitHub Actions** to produce an AI-agent **dossier**
for its repos and, optionally, share a redacted summary with the SettleTop registry.

**Your source never leaves your runner.** The scan runs in a sandboxed container
with no network. Only a redacted "share view" is uploaded, and only when you set
`share: registry` — never source, code snippets, secret values, or secret locations.

## Before you can use it

1. Install the **SettleTop AI Dossier** GitHub App on your org.
2. Connect the org to your SettleTop account at <https://registry.settletop.com>.

The App only reads repo **metadata** + which workflow runs happened. It never runs
code or reads your source.

## How to use it

Add a small workflow to your repo, e.g. `.github/workflows/dossier.yml`:

```yaml
name: Dossier
on:
  push: { branches: [main] }
  schedule: [{ cron: "0 6 * * 1" }]
  workflow_dispatch: {}
jobs:
  dossier:
    uses: SettleTop-Inc/settletop-dossier-workflow/.github/workflows/private-door.yml@v1
    permissions:
      id-token: write         # licence + share
      contents: write         # keep: release
      checks: write           # keep: checks
      security-events: write  # keep: security-tab
      actions: write          # cleanup
    with:
      keep: summary,artifact
      share: registry
      registry-ingest-url: https://registry.settletop.com/api/ingest/scan
      registry-audience: https://ingest.settletop.com
      engine-image: ghcr.io/settletop-inc/dossier-engine@sha256:<digest>  # see Releases
```

Pin to a tag (`@v1`) or a commit SHA — not `@main`.

## Inputs

| input | default | what it does |
|-------|---------|--------------|
| `keep` | `summary,artifact` | Where results are kept: `summary`, `checks`, `artifact`, `security-tab`, `release`, or `none`. |
| `keep-retention-days` | `0` | Retention for the `artifact` output (0 = repo/org default). |
| `share` | `none` | `registry` uploads the share view; `none` skips it. |
| `max-share-bytes` | `5242880` | Share-view size cap; over it, nothing is uploaded. |
| `engine-image` | *(placeholder)* | The licence-gated engine image, pinned by digest. |
| `engine-commit` | `""` | Engine source commit recorded in `run_meta`. |
| `engine-memory` | `4g` | Memory limit for the engine container. |
| `licence-url` | *(placeholder)* | SettleTop licence service endpoint. |
| `licence-audience` | *(placeholder)* | Fixed `aud` for the licence token. |
| `registry-ingest-url` | *(placeholder)* | Registry ingest endpoint (`POST /api/ingest/scan`). |
| `registry-audience` | *(placeholder)* | Fixed `aud` for the ingest token. |

Outputs: `shared` (bool) and `share-sha256`.

## Permissions

The caller grants the **union** of what the jobs need (see the example). Each job
then uses only its own slice; `cleanup` uses your own token, never the App.

## What the jobs do

`[fork guard] → licence → check → keep* / share → cleanup`

- **licence** — trades GitHub's identity token for a short-lived licence (copy-protection only; contacts SettleTop on every run).
- **check** — runs the sandboxed engine image (no network, non-root, read-only FS); emits the share view, BOM, SARIF, findings, HTML summary.
- **keep*** — one job per `keep` output, each with only the permission that output needs.
- **share** — validates the report and POSTs the share view to the registry (only with `share: registry`).
- **cleanup** — deletes the transfer artifacts (`if: always()`).

## Status

**Not yet runnable end-to-end:** the `engine-image` is not published to GHCR yet.
Once it is, set its digest in `engine-image` (and the licence-service URLs) and it works.
