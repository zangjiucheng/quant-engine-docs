# quant-engine-docs

Public docs site for the `quant-engine` C++ quant backtest + research
engine (source is private). Published to GitHub Pages from the `main`
branch via `.github/workflows/gh-pages.yml`.

Interested in internal testing access? Reach out to
**jiucheng.zang@proton.me**.

**Do not edit files here directly** — this repo is force-pushed from the
upstream private repo by an internal publish script. Any local commits
will be overwritten on the next publish.

## Layout

- `docs/` — Markdown sources (mkdocs Material)
- `mkdocs.yml` — site config
- `docs/requirements.txt` — Python deps for the build
- `.github/workflows/gh-pages.yml` — Pages publish workflow

## Build locally

```bash
pip install -r docs/requirements.txt
mkdocs serve            # live preview at http://127.0.0.1:8000
mkdocs build --strict   # one-shot build into ./site
```
