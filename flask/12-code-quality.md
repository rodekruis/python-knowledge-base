# 12 — Code Quality

> **Shared foundation:** See [../shared/code-quality.md](../shared/code-quality.md) for the canonical ruff config, type checking (ty/mypy), pre-commit setup, naming conventions, docstring style, anti-patterns, and Makefile template.

This addendum covers **Flask-specific** code quality patterns only.

---

## Type Hints in Flask

Add type annotations to all function signatures — non-negotiable for maintainability:

```python
# ❌ Bad — no type hints, unclear API
def createBatchesRandomSampling(df, min_batchsize, batchsize, lastBatch, district, cookie, project):
    ...

# ✅ Good — self-documenting with type hints
def create_batches_random_sampling(
    df: pd.DataFrame,
    allow_smaller_last: bool,
    batch_size: int,
    last_batch_number: int,
    district: str,
    access_token: str,
    project_id: int,
) -> list[str]:
    ...
```

### Flask-Specific Type Patterns

```python
from flask import Flask, Response
from werkzeug.wrappers import Response as WerkzeugResponse

def create_app(config_class: type = Config) -> Flask:
    ...

@main_bp.route("/users")
def list_users() -> str:  # Returns rendered template
    return render_template("users.html", users=users)

@main_bp.route("/api/data")
def get_data() -> tuple[dict, int]:  # JSON + status code
    return {"items": items}, 200
```

---

## Don't Mix Presentation and Business Logic

This is the most common anti-pattern in Flask apps:

```python
# ❌ Bad — Flask import in a service module
from flask import render_template

def create_batches_random_sampling(...):
    if min_batchsize not in ("yes", "no"):
        return render_template("error.html", error="Invalid input")

# ✅ Good — raise, let the route handler deal with presentation
def create_batches_random_sampling(...):
    if min_batchsize not in ("yes", "no"):
        raise ValueError("min_batchsize must be 'yes' or 'no'")
```

Route handlers call services; services never import from `flask`.

---

## Per-File Ignores for Tests

Allow `assert` statements in test files (needed for pytest):

```toml
[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101"]
```

