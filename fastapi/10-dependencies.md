# 10 — Dependency Management

> **Shared foundation:** See [../shared/dependencies.md](../shared/dependencies.md) for the canonical pyproject.toml template, uv commands, lockfile strategy, version pinning, Dependabot config, and migration guide.

This addendum covers **FastAPI-specific** dependency patterns only.

---

## Typical FastAPI Dependencies

```toml
[project]
name = "my-app"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi",
    "uvicorn",
    "gunicorn",
    "python-dotenv",
    "pydantic-settings",
    "httpx",
]

[dependency-groups]
dev = [
    "pytest>=8.0",
    "pytest-cov>=6.0",
    "httpx>=0.27",
    "ruff>=0.8",
    "ty>=0.0.1a1",
]
```

### Common FastAPI Extensions

| Package | Purpose |
|---------|---------|
| `pydantic-settings` | Typed configuration from env vars |
| `httpx` | Async HTTP client (also used for `TestClient`) |
| `sqlalchemy[asyncio]` | Async database ORM |
| `alembic` | Database migrations |
| `uvicorn` | ASGI server (development) |
| `gunicorn` | Process manager (production, with UvicornWorker) |
| `python-multipart` | File uploads / form data |

### FastAPI-Specific Dev Tools

| Package | Purpose |
|---------|---------|
| `httpx` | Required for `TestClient` (async support) |
| `import-linter` | Enforce API → Service → Domain layers |

---

## Version Pinning: Unpinned for FastAPI Apps

FastAPI projects often use **unpinned** direct dependencies (`"fastapi"` instead of `"fastapi>=0.115,<1"`), relying entirely on the lockfile for reproducibility. This is acceptable because:

- `uv.lock` guarantees exact versions in CI and Docker
- FastAPI's ecosystem moves fast; upper bounds cause friction
- The lockfile is the source of truth for builds

If you prefer explicit bounds (recommended for libraries), see the shared [pinning strategy](../shared/dependencies.md#version-pinning-strategy).

---

## OneDrive Compatibility

If your project lives on OneDrive (common on Windows), hardlinks fail:

```powershell
# Set permanently in PowerShell profile
[Environment]::SetEnvironmentVariable("UV_LINK_MODE", "copy", "User")
```

---

## Cleaning Up Unused Dependencies

Periodically audit for packages left over from experiments:

```bash
uv pip list  # Compare against actual imports in src/
```

Common culprits: `chromadb`, `tiktoken`, `python-jose`, `passlib` left from prototyping.

