# bbstage documentation site

MkDocs-Material documentation for the **bbstage** ADIOS2 engine (fork of
ADIOS2 with node-local staging + async drain). Style reference: the GSI
Virgo user guide (MkDocs-Material as well).

## Content sources (provenance)

| page | derived from |
|---|---|
| `docs/index, quickstart, parameters, semantics` | `BBSTAGE.md` at fork root (github.com/denisbertini/ADIOS2) |
| `docs/tuning, methodology, diagnostics, faq` | the bbphase measurement campaign (`CAMPAIGN_RESULTS.md`, job IDs cited inline) |

Keep in sync: parameter changes must land in BOTH the fork's
`BBSTAGE.md` + `docs/user_guide/.../bbstage.rst` (Sphinx, ships with the
fork) and `docs/parameters.md` (this site).

## Local preview

```bash
python3 -m venv .venv && . .venv/bin/activate
pip install mkdocs-material
mkdocs serve          # http://127.0.0.1:8000
```

## Publishing (no server: GitHub Pages + Actions)

1. Push this repo to GitHub.
2. Repo **Settings → Pages → Build source: GitHub Actions**.
3. Every push to `main` builds and deploys via
   `.github/workflows/docs.yml` →
   `https://<user>.github.io/bbstage-docs/`.
