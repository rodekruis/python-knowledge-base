# Docker

## Key Principles

1. **Layer ordering** — dependencies before code (code changes don't bust the dep-install cache)
2. **`--no-dev`** — no test/lint tools in production
3. **`--frozen`** — fail if the lockfile is stale (never resolve in CI/Docker)
4. **`--no-editable`** — install as a proper package, not editable
5. **No secrets in the image** — `.env` is never copied; secrets come from env vars at runtime
6. **No tests in production** — exclude `tests/` to reduce image size and avoid information leaks
7. **Clean up apt lists** — smaller image when installing system packages

---

## Base Dockerfile Pattern

```dockerfile
FROM python:3.12-slim

# System dependencies (only if needed at runtime)
# RUN apt-get update && apt-get install -y --no-install-recommends \
#     libpq-dev \
#     && rm -rf /var/lib/apt/lists/*

# Install uv for fast dependency management
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

WORKDIR /app

# Install dependencies first (cache layer)
COPY pyproject.toml uv.lock ./
RUN uv sync --no-editable --no-dev --frozen

# Copy application code
COPY src/ src/

EXPOSE 8000
# CMD differs per framework — see below
```

### Framework-Specific CMD

| Framework | Server | CMD |
|-----------|--------|-----|
| Flask (WSGI) | Gunicorn | `CMD [".venv/bin/gunicorn", "wsgi:app", "--bind", "0.0.0.0:8000", "--workers", "2"]` |
| FastAPI (ASGI) | Gunicorn + Uvicorn | `CMD [".venv/bin/gunicorn", "myapp.main:app", "--worker-class", "uvicorn.workers.UvicornWorker", "--bind", "0.0.0.0:8000", "--workers", "2"]` |

---

## .dockerignore

Always include a `.dockerignore` to keep the build context small:

```dockerignore
.git/
.github/
.venv/
__pycache__/
*.pyc
.env
.env.*
tests/
docs/
*.md
!README.md
.pytest_cache/
.mypy_cache/
.ruff_cache/
.vscode/
node_modules/
```

---

## Development vs Production

| Concern | Development | Production |
|---------|------------|------------|
| Server | `flask run` / `uvicorn --reload` | Gunicorn (with appropriate worker class) |
| Workers | 1 | 2–4 (`2 * CPU_CORES + 1`) |
| Dependencies | `uv sync` (includes dev) | `uv sync --no-dev --frozen` |
| Env vars | `.env` file | Azure App Service / Key Vault |
| Image | Not needed | Multi-layer optimized |
| Reload | Auto-reload on save | Never |

---

## Health Check

Add a Docker HEALTHCHECK so orchestrators know when the container is ready:

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1
```

This uses Python's stdlib — no need to install curl in the slim image.

---

## Entrypoint Script (Complex Apps)

For apps that need pre-start tasks (database migrations, cache warming):

```sh
#!/usr/bin/env sh
set -eu

# Run migrations
.venv/bin/python -m myapp.cli.migrate

# Start the server
exec .venv/bin/gunicorn myapp.main:app \
    --worker-class uvicorn.workers.UvicornWorker \
    --bind 0.0.0.0:8000 \
    --workers "${GUNICORN_WORKERS:-2}" \
    --timeout 120
```

```dockerfile
COPY entrypoint.sh ./
RUN chmod +x entrypoint.sh
ENTRYPOINT ["./entrypoint.sh"]
```

---

## Gunicorn Configuration (Optional)

For complex setups, use a config file instead of CLI flags:

```python
# gunicorn.conf.py
import os

bind = f"0.0.0.0:{os.environ.get('PORT', '8000')}"
workers = int(os.environ.get("GUNICORN_WORKERS", "2"))
timeout = 120
accesslog = "-"  # stdout
errorlog = "-"   # stderr
```

```dockerfile
CMD [".venv/bin/gunicorn", "wsgi:app", "--config", "gunicorn.conf.py"]
```

---

## Azure App Service Notes

- Azure injects the `PORT` environment variable — bind to it:
  ```python
  bind = f"0.0.0.0:{os.environ.get('PORT', '8000')}"
  ```
- For non-Docker deployments, Azure uses Gunicorn by default and looks for `wsgi.py` or `app.py`
- Set the startup command explicitly in Azure Portal or via `az webapp config`
- For Docker deployments, Azure pulls from your container registry (GHCR, ACR)

---

## Never Run Dev Servers in Production

❌ Bad:
```python
app.run(host="0.0.0.0", port=8000)  # Single-threaded, no worker management
uvicorn.run(app, host="0.0.0.0")    # No workers, no process management
```

✅ Good:
```bash
gunicorn wsgi:app --bind 0.0.0.0:8000 --workers 2           # Flask
gunicorn app:app --worker-class uvicorn.workers.UvicornWorker --workers 2  # FastAPI
```

Docker/production must always use Gunicorn for process management, graceful restarts, and worker supervision.
