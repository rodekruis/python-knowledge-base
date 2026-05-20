# 08 — Docker

> **Shared foundation:** See [../shared/docker.md](../shared/docker.md) for universal Dockerfile principles, .dockerignore template, layer caching strategy, dev vs prod comparison, health checks, and entrypoint scripts.

This addendum covers **Flask-specific** Docker patterns only.

---

## Production Dockerfile (Flask / WSGI)

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install uv for fast dependency management
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

# Install dependencies first (cache layer)
COPY pyproject.toml uv.lock ./
RUN uv sync --no-editable --no-dev --frozen

# Copy application code
COPY src/ src/
COPY wsgi.py ./

EXPOSE 8000

CMD [".venv/bin/gunicorn", "wsgi:app", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "2", \
     "--timeout", "120"]
```

## Simplified Dockerfile (Flat Structure)

For small apps without `src/` layout:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

COPY pyproject.toml uv.lock ./
RUN uv sync --no-editable --no-dev --frozen

COPY app.py wsgi.py utils.py ./
COPY templates/ templates/
COPY static/ static/

EXPOSE 8000

CMD [".venv/bin/gunicorn", "wsgi:app", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "2", \
     "--timeout", "120"]
```

---

## Gunicorn Configuration

Flask's built-in server (`flask run` / `app.run()`) is **development only**. In production, always use a **[WSGI](README.md#glossary)** server like **[Gunicorn](README.md#glossary)**:

```bash
gunicorn wsgi:app --bind 0.0.0.0:8000 --workers 2 --timeout 120
```

### Worker count

Rule of thumb: `2 * CPU_CORES + 1`. For Azure App Service B1 (1 core), use 2-3 workers.

### Configuration file (optional)

```python
# gunicorn.conf.py
import os

bind = f"0.0.0.0:{os.environ.get('PORT', '8000')}"
workers = int(os.environ.get("GUNICORN_WORKERS", "2"))
timeout = 120
accesslog = "-"  # stdout
errorlog = "-"   # stderr
```

---

## Azure App Service Specific

Azure App Service for Linux uses Gunicorn by default. It looks for:
1. A custom startup command (set in Azure Portal or `az webapp config`)
2. Or a file named `wsgi.py` or `app.py` with an `app` variable

Set the startup command explicitly:
```bash
gunicorn --bind=0.0.0.0 --timeout 120 wsgi:app
```

Azure injects the `PORT` environment variable — bind to it in production.

---

## Never Run Flask's Dev Server in Production

❌ Bad:
```python
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)  # Single-threaded, no worker management
```

✅ Good:
```bash
gunicorn wsgi:app --bind 0.0.0.0:8000 --workers 2
```

