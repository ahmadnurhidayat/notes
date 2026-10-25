# Ahmad's Notes

Working notes on cloud-native infrastructure — the quick things I look *up*, not read cover to cover. Built with [Zensical](https://zensical.org) (the modern static site generator from the [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) team).

**Live:** [notes.beyondyou.my.id](https://notes.beyondyou.my.id) · **Longer writing:** [ahmadnurhidayat.github.io](https://ahmadnurhidayat.github.io)

## What's here

This is the raw reference layer of a hub-and-spoke setup. Two kinds of notes:

- **TIL** — one small, dated, tagged thing I learned. A running stream.
- **Guides** — full how-tos and runbooks (Helm installs, observability, GitOps, platform engineering).

Focus areas: Kubernetes · Networking · GitOps · Observability · CNCF · Cloud (GCP / AWS).

Polished, opinionated essays live on the personal site; this site is lookup-first.

## Tech stack

| Layer | Technology |
|---|---|
| **Generator** | [Zensical](https://zensical.org) (`modern` theme variant) |
| **Config** | `mkdocs.yml` (MkDocs-compatible, so Material for MkDocs is a fallback) |
| **Search** | Built-in (press `/`) |
| **Markdown** | Python Markdown + PyMdown Extensions (admonitions, tabs, code, mermaid) |
| **Deploy** | GitHub Actions → GitHub Pages |

## Project structure

```
notes/
├── mkdocs.yml                 # site config: theme, nav, search, tags, markdown extensions
├── docs/
│   ├── CNAME                  # custom domain (notes.beyondyou.my.id)
│   ├── index.md               # landing page
│   ├── til/                   # Today-I-Learned entries (dated, tagged)
│   │   └── index.md
│   └── guides/                # runbooks and how-tos, grouped by area
│       ├── index.md
│       └── observability/
│           └── loki-helm-production.md
└── .github/workflows/docs.yml # build + deploy to GitHub Pages
```

## Local development

### Prerequisites

- Python 3.10+
- A Python package manager (`pip`)

### Setup

```bash
# one-time: create a virtual environment and install Zensical
python3 -m venv .venv
source .venv/bin/activate
pip install zensical
```

### Common commands

All commands assume the virtual environment is active (`source .venv/bin/activate`).

| Command | Action |
|---|---|
| `zensical serve` | Start the local dev server with live reload |
| `zensical build --clean` | Build the static site into `site/` |
| `zensical new <dir>` | Scaffold a new project (not needed here) |

Preview locally, then open the printed URL (default `http://localhost:8000`).

## Adding content

### A TIL entry

Create a new `.md` file in `docs/til/` and add it to the `nav` in `mkdocs.yml`:

```markdown
---
date: 2026-10-25
tags:
  - kubernetes
  - networking
---

# Short, specific title

The problem, the fix, and *why* it bites. Keep it to one idea.
```

### A guide

Create a new `.md` file under `docs/guides/<area>/` and add it to the `nav`:

```markdown
---
tags:
  - observability
  - helm
---

# Guide title

Step-by-step runbook. Use admonitions, content tabs, and code blocks freely.
```

Both post types are tagged; the `tags` plugin builds filterable indexes.

## Deployment

Pushes to `main`/`master` trigger `.github/workflows/docs.yml`, which:

1. Installs Zensical (`pip install zensical`)
2. Builds the site (`zensical build --clean`)
3. Uploads `site/` and deploys to GitHub Pages

The custom domain is set via `docs/CNAME` (copied into the build output). Point a `notes.beyondyou.my.id` CNAME DNS record at GitHub Pages to go live.

## Notes on Zensical

Zensical is in active beta (pre-1.0). Because this project uses `mkdocs.yml`, it stays compatible with [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) as a fallback — `pip install mkdocs-material && mkdocs build` builds the same sources if ever needed.

## License

MIT
