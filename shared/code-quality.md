# Code Quality

## Ruff (Linter + Formatter)

**[Ruff](https://docs.astral.sh/ruff/)** replaces flake8, isort, pycodestyle, and Black in a single tool.

### Configuration (in pyproject.toml)

```toml
[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes
    "I",    # isort (import sorting)
    "UP",   # pyupgrade (modernize syntax)
    "B",    # bugbear (common pitfalls)
    "SIM",  # simplify (unnecessary complexity)
    "T20",  # flake8-print (no print() in production)
    "N",    # pep8-naming
    "RET",  # flake8-return (consistent returns)
    "PTH",  # flake8-use-pathlib
]

[tool.ruff.lint.isort]
known-first-party = ["my_app"]

[tool.ruff.format]
quote-style = "double"
```

### Commands

```bash
ruff check .            # Lint
ruff check . --fix      # Lint + auto-fix
ruff format .           # Format
ruff format --check .   # Check format (CI)
```

---

## Type Checking

### ty (Recommended for New Projects)

**[ty](https://docs.astral.sh/ty/)** is Astral's new type checker — fast, zero config, designed to work alongside ruff:

```bash
uv add --dev ty
uv run ty check
```

> **Note:** ty is in early development (as of 2025). It may not yet cover all edge cases that mypy handles.

### mypy (Established Alternative)

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_configs = true
```

```bash
uv run mypy src/
```

### Recommendation

- **New projects** → try ty first; fall back to mypy if you hit gaps
- **Existing projects** → keep mypy if already configured
- **CI** → run whichever you chose; don't run both

---

## Pre-commit

Automate quality checks on every commit:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.6
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
        args: [--maxkb=500]
```

### Setup

```bash
uv add --dev pre-commit
uv run pre-commit install          # Activate hooks
uv run pre-commit run --all-files  # Run on entire repo
```

---

## Naming Conventions

| Element | Style | Example |
|---------|-------|---------|
| Module / package | `snake_case` | `user_service.py` |
| Class | `PascalCase` | `UserRepository` |
| Function / method | `snake_case` | `get_active_users()` |
| Constant | `UPPER_SNAKE` | `MAX_RETRIES` |
| Private | leading `_` | `_validate_input()` |
| Type alias | `PascalCase` | `UserId = int` |
| Test function | `test_<what>_<condition>` | `test_login_invalid_password()` |

---

## Docstrings (Google Style)

```python
def fetch_data(url: str, *, timeout: int = 30) -> dict:
    """Fetch JSON data from a remote endpoint.

    Args:
        url: The endpoint URL to fetch from.
        timeout: Request timeout in seconds.

    Returns:
        Parsed JSON response as a dictionary.

    Raises:
        httpx.HTTPStatusError: If the response status is 4xx/5xx.
    """
```

### Where to Write Docstrings

- ✅ All public functions, classes, and methods
- ✅ Module-level docstring for non-trivial modules
- ❌ Skip for obvious one-liners (`__init__` with simple assignment)
- ❌ Skip for private helpers if the name is self-documenting

---

## Anti-Patterns to Avoid

| Anti-pattern | Fix |
|-------------|-----|
| `from module import *` | Explicit imports only |
| Bare `except:` | Catch specific exceptions |
| Mutable default args (`def f(x=[])`) | Use `None` + conditional |
| `print()` for logging | Use `logging` module |
| God functions (>50 lines) | Extract helpers |
| Deep nesting (>3 levels) | Early returns / guard clauses |
| String concatenation for SQL | Parameterized queries |
| Ignoring return values | Assign or explicitly discard |

---

## Makefile (Optional Convenience)

```makefile
.PHONY: lint format test type-check all

lint:
	uv run ruff check .

format:
	uv run ruff format .

test:
	uv run pytest --cov=src --cov-report=term-missing

type-check:
	uv run ty check

all: lint format type-check test
```

---

## Import Linter (Layered Architectures)

For projects with strict layer boundaries (API → Service → Repository):

```toml
# pyproject.toml
[tool.importlinter]
root_packages = ["my_app"]

[[tool.importlinter.contracts]]
name = "Layers"
type = "layers"
layers = [
    "my_app.api",
    "my_app.services",
    "my_app.repositories",
]
```

```bash
uv add --dev import-linter
uv run lint-imports
```

---

## CI Integration

Add quality checks to your GitHub Actions workflow (see [ci-cd.md](ci-cd.md)):

```yaml
- name: Lint
  run: uv run ruff check .

- name: Format check
  run: uv run ruff format --check .

- name: Type check
  run: uv run ty check

- name: Test
  run: uv run pytest --cov=src --cov-report=term-missing
```

All checks must pass before merge. No exceptions.
