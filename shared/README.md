# Shared Python Best Practices

Framework-agnostic tooling, CI/CD, and quality standards that apply to **all** Python projects (Flask, FastAPI, ETL pipelines, etc.).

## Guide Structure

| File | Topic |
|----------|-------|
| [dependencies.md](dependencies.md) | Dependency management with uv, pyproject.toml template, lockfiles, Dependabot |
| [ci-cd.md](ci-cd.md) | GitHub Actions workflows, environment secrets, branch protection |
| [code-quality.md](code-quality.md) | Ruff, type checking, pre-commit, naming conventions, docstrings |
| [logging.md](logging.md) | Structured logging, log levels, what to log, Azure Monitor |
| [docker.md](docker.md) | Dockerfile principles, .dockerignore, layer caching, dev vs prod |

## Relationship to Framework Guides

Each framework guide (Flask, FastAPI, etc.) has thin addenda that cover only
framework-specific deviations:

- **Flask** — WSGI/Gunicorn CMD, `@after_request` logging, zip-deploy CI job
- **FastAPI** — ASGI/UvicornWorker CMD, middleware logging, Docker-deploy CI job, import-linter

The shared docs define the **canonical defaults**; framework addenda override or
extend only where necessary.

## Glossary

Framework-agnostic terms used across all Python projects:

| Term | Definition |
|------|------------|
| **Gunicorn** | A production-grade Python HTTP server that manages multiple worker processes for concurrency, graceful restarts, and process supervision. Used with WSGI apps (Flask) directly, or with `UvicornWorker` for ASGI apps (FastAPI). Never use framework dev servers in production. See [docker.md](docker.md). |
| **Lockfile (`uv.lock`)** | A file that pins every direct and transitive dependency to an exact version, ensuring deterministic, reproducible installs across dev machines, CI, and Docker. Always commit it; use `--frozen` in CI/Docker. See [dependencies.md](dependencies.md). |
| **OpenTelemetry** | A vendor-neutral observability framework for collecting traces, metrics, and logs. Used with `azure-monitor-opentelemetry` to export data to Azure Application Insights. Replaces the deprecated `opencensus` libraries. See [logging.md](logging.md). |
| **Pre-commit** | A framework for managing and running git hook scripts automatically on every commit. Ensures code is linted and formatted before it reaches the repository. See [code-quality.md](code-quality.md). |
| **Ruff** | An extremely fast Python linter and formatter that replaces flake8, isort, pycodestyle, and Black in a single tool. Configured in `pyproject.toml`. See [code-quality.md](code-quality.md). |
| **uv** | A fast Python package manager that replaces pip, pip-tools, and virtualenv. Provides lockfile support, deterministic resolution, and Python version management in one tool. See [dependencies.md](dependencies.md). |
