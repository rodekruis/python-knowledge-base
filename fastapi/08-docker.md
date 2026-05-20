# 08 — Docker

> **Shared foundation:** See [../shared/docker.md](../shared/docker.md) for universal Dockerfile principles, .dockerignore template, layer caching strategy, dev vs prod comparison, health checks, and entrypoint scripts.

This addendum covers **FastAPI-specific** Docker patterns only.

---

## Production Dockerfile (FastAPI / ASGI)

```dockerfile
FROM python:3.12-slim

# System dependencies (only what's needed at runtime)
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

WORKDIR /app

# Install dependencies first (cache layer)
COPY pyproject.toml uv.lock ./
RUN uv sync --no-editable --no-dev --frozen

# Copy application code
COPY src/ src/
COPY alembic.ini ./
COPY alembic/ alembic/
COPY entrypoint.sh ./
RUN chmod +x entrypoint.sh

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

ENTRYPOINT ["./entrypoint.sh"]
```

---

## Entrypoint Script

For apps with database migrations:

```sh
#!/usr/bin/env sh
set -eu

# Run migrations before binding the port
.venv/bin/python -m alembic upgrade head

# Start the ASGI server
exec .venv/bin/gunicorn myapp.main:app \
    --worker-class uvicorn.workers.UvicornWorker \
    --bind 0.0.0.0:8000 \
    --workers "${GUNICORN_WORKERS:-2}" \
    --timeout 120
```

For simple apps without migrations:

```dockerfile
CMD [".venv/bin/gunicorn", "myapp.main:app", \
     "--worker-class", "uvicorn.workers.UvicornWorker", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "2", \
     "--timeout", "120"]
```

---

## Simplified Dockerfile (Small Apps)

For services without `src/` layout:

```dockerfile
FROM python:3.12-slim

COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

WORKDIR /app

COPY pyproject.toml uv.lock ./
RUN uv sync --no-dev --frozen

COPY routes/ routes/
COPY utils/ utils/
COPY main.py ./

ENV PORT=8000
EXPOSE 8000
CMD ["uv", "run", "python", "main.py"]
```

---

## Health Check Endpoint

```python
@router.get("/health", tags=["system"])
async def health():
    return {"status": "ok"}
```

---

## Never Run Uvicorn Directly in Production

❌ Bad:
```python
uvicorn.run(app, host="0.0.0.0", port=8000)  # No workers, no process management
```

✅ Good:
```bash
gunicorn myapp.main:app --worker-class uvicorn.workers.UvicornWorker --workers 2
```

**[Gunicorn + UvicornWorker](README.md#glossary)** provides process management, graceful restarts, and multi-worker support.
