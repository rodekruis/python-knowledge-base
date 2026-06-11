# Copilot Instructions — Python Knowledge Base

This repository documents NLRC 510's Python best practices. It contains topic guides plus a single
consolidated `copilot-instructions.md` per framework that downstream projects copy into their own repos.

## Repository Layout

```
shared/                 # framework-agnostic standards (dependencies, ci-cd, code-quality, logging, docker)
fastapi/                # FastAPI guide (01-*.md … 12-*.md) + copilot-instructions.md
flask/                  # Flask guide (01-*.md … 12-*.md) + copilot-instructions.md
etl-pipelines/          # ETL guide (01-*.md … 13-*.md) + copilot-instructions.md
data-analysis/          # Jupyter/notebook guide (01-*.md … 10-*.md) + copilot-instructions.md
```

Each framework folder's `copilot-instructions.md` is a **concise consolidation** of that folder's numbered
guides **plus** the relevant `shared/` standards. These files are meant to be dropped directly into a new
project at `.github/copilot-instructions.md`.

## CRITICAL: Keep consolidated instructions in sync

The per-folder `copilot-instructions.md` files are **derived artifacts**. They must stay consistent with the
source guides they summarize. Whenever you change content, propagate it:

- **When you update any guide file in a framework folder `X/`** (e.g. `fastapi/03-routes-and-schemas.md`),
  review and update **`X/copilot-instructions.md`** so it still reflects the guidance.
- **When you update anything in `shared/`** (e.g. `shared/docker.md`), review and update the
  `copilot-instructions.md` in **every** framework folder that relies on that shared standard
  (`fastapi/`, `flask/`, `etl-pipelines/`, `data-analysis/`), since they all consolidate shared content.
- **When you add a new framework folder**, create its `copilot-instructions.md` following the same structure
  and add it to the table in the root `README.md`.

After editing, **cross-check the four `copilot-instructions.md` files for consistency** on shared topics
(Python version, ruff config, Docker base image, uv usage, observability stack, license). Differences must be
intentional and explicitly flagged — for example:

- `data-analysis/` uses **conda/mamba**, not uv (compiled geospatial libraries). This deviation is documented.
- **Flask** deploys via **zip deploy**; **FastAPI** and **ETL** deploy via **Docker**.

## Editing Conventions

- Keep `copilot-instructions.md` files **concise and actionable** — they are loaded into an agent's context.
  Favor short imperative rules and minimal illustrative snippets over long prose.
- Keep shared standards canonical in `shared/`; framework guides and their consolidated files should reference
  or briefly restate shared rules, not contradict them.
- Do not duplicate full content from numbered guides into `copilot-instructions.md` — summarize the decisions.
- Preserve existing relative-link style and markdown table formatting in the guides.