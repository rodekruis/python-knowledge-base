# 10 — Dependencies

> **Shared foundation:** See [../shared/dependencies.md](../shared/dependencies.md) for the canonical pyproject.toml template, uv commands, lockfile strategy, version pinning, Dependabot config, and migration guide.

This addendum covers **Flask-specific** dependency patterns only.

---

## Typical Flask Dependencies

```toml
[project]
name = "my-flask-app"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "flask>=3.1,<4",
    "flask-wtf>=1.2,<2",
    "gunicorn>=22,<23",
    "python-dotenv>=1,<2",
]

[dependency-groups]
dev = [
    "pytest>=8.0",
    "pytest-cov>=6.0",
    "ruff>=0.8",
    "ty>=0.0.1a1",
]
```

### Common Flask Extensions

| Package | Purpose |
|---------|---------|
| `flask-wtf` | Form handling with CSRF protection |
| `flask-login` | Session-based authentication |
| `flask-sqlalchemy` | SQLAlchemy integration |
| `flask-migrate` | Alembic migrations for Flask-SQLAlchemy |
| `gunicorn` | Production WSGI server |
| `python-dotenv` | Load `.env` / `.flaskenv` files |

### Flask-Specific Dev Tools

| Package | Purpose |
|---------|---------|
| `responses` | Mock `requests` library calls |
| `pytest-mock` | General mocking |
| `freezegun` | Time-dependent test control |

---

## .flaskenv vs .env

Flask uses `python-dotenv` to load environment files:

| File | Purpose | Commit? |
|------|---------|---------|
| `.flaskenv` | Non-secret Flask config (`FLASK_APP`, `FLASK_DEBUG`) | ✅ Yes |
| `.env` | Secrets (`SECRET_KEY`, API keys) | ❌ Never |

```bash
# .flaskenv (committed)
FLASK_APP=src/my_app
FLASK_DEBUG=1
```

