# 13 — Dependencies and Code Quality

> **Shared foundation:** See [../shared/dependencies.md](../shared/dependencies.md) for the canonical pyproject.toml template, uv commands, lockfile strategy, and version pinning. See [../shared/code-quality.md](../shared/code-quality.md) for ruff config, pre-commit, naming conventions, and type checking.

This document covers **pipeline-specific** tooling: additional quality tools (deptry, vulture, pyright), the quality runner script pattern, poetry support for existing projects, and pipeline-specific dependency patterns.

---

## Package Manager: uv or poetry

Use [uv](https://docs.astral.sh/uv/) (recommended) or [poetry](https://python-poetry.org/). Both enforce lockfiles and use `pyproject.toml`.

**Our recommendation**: Use **uv** for new projects (10-100x faster installs, smaller lockfiles, better cross-platform support). Keep **poetry** if your team already uses it — it works fine. Don't mix both in the same project.

For uv commands, see [../shared/dependencies.md](../shared/dependencies.md#essential-commands).

### poetry (existing projects only)

```bash
poetry install                    # Initial setup
poetry add httpx                  # Add a dependency
poetry update                     # Update all deps
poetry install --no-interaction   # CI (from lockfile)
```

## Pipeline pyproject.toml

Start from the [shared template](../shared/dependencies.md#pyprojecttoml-template) and add pipeline-specific dependencies and tooling:

```toml
[project]
name = "my-pipeline"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "click>=8.1",
    "pyyaml>=6.0",
    "requests>=2.31",
    "pydantic>=2.0",        # If you need schema validation
]

[project.scripts]
run-pipeline = "my_pipeline.infra.run_forecasts:run_forecasts"

[dependency-groups]
dev = [
    "pytest>=8.0",
    "pytest-cov>=5.0",
    "ruff>=0.8",
    "pyright>=1.1",
    "deptry>=0.20",
    "vulture>=2.11",
    "responses>=0.25",      # HTTP mocking
]

[tool.pyright]
pythonVersion = "3.12"
typeCheckingMode = "standard"
reportMissingImports = true

[tool.pytest.ini_options]
testpaths = ["tests"]
markers = [
    "integration: marks tests as integration tests (deselect with '-m not integration')",
]
```

### Pipeline-specific deviations from shared config

| Setting | Shared default | Pipeline override | Reason |
|---------|---------------|-------------------|--------|
| Type checker | ty | pyright | Better Protocol support, strict mode |
| Extra dev deps | — | deptry, vulture | Dependency hygiene + dead code detection |
| Extra lint rules | — | `"RUF"` (ruff-specific) | Catches pipeline anti-patterns |

For ruff and pre-commit config, use the [shared canonical config](../shared/code-quality.md#ruff-linter--formatter). Add `"RUF"` to the `select` list for pipelines.

---

## Code Quality Toolchain

Run all quality checks with a single script:

### `scripts/check.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "=== Ruff: lint ==="
uv run ruff check .

echo "=== Ruff: format ==="
uv run ruff format --check .

echo "=== Pyright: type check ==="
uv run pyright

echo "=== Deptry: unused/missing deps ==="
uv run deptry .

echo "=== Vulture: dead code ==="
uv run vulture src/ --min-confidence 80

echo "=== All checks passed ==="
```

Or as a Python script (Approach A pattern):

### `scripts/python-knip.py`

```python
#!/usr/bin/env python3
"""Run all code quality checks. Exit non-zero on any failure."""

import subprocess
import sys

CHECKS = [
    ("Ruff lint", ["uv", "run", "ruff", "check", "."]),
    ("Ruff format", ["uv", "run", "ruff", "format", "--check", "."]),
    ("Pyright", ["uv", "run", "pyright"]),
    ("Deptry", ["uv", "run", "deptry", "."]),
    ("Vulture", ["uv", "run", "vulture", "src/", "--min-confidence", "80"]),
]

def main():
    failed = []
    for name, cmd in CHECKS:
        print(f"\n{'='*60}")
        print(f"  {name}")
        print(f"{'='*60}")
        result = subprocess.run(cmd)
        if result.returncode != 0:
            failed.append(name)

    if failed:
        print(f"\n❌ Failed checks: {', '.join(failed)}")
        sys.exit(1)
    print("\n✅ All checks passed")

if __name__ == "__main__":
    main()
```

## What Each Tool Does

| Tool | Purpose | Why |
|------|---------|-----|
| **ruff** | Linting + formatting | Replaces flake8, isort, black. 10-100x faster. One tool. |
| **pyright** | Static type checking | Catches type errors before runtime. Strict mode available. |
| **deptry** | Dependency hygiene | Finds unused deps, missing deps, transitive deps used directly. |
| **vulture** | Dead code detection | Finds unused functions, variables, imports. |
| **pytest** | Testing | Standard Python test framework. |
| **pytest-cov** | Coverage reporting | Ensures tests cover critical paths. |

## Pre-commit Hooks

Recommended — catch issues before they reach CI. Use pre-commit in all projects.

## Pre-commit Hooks

Use the [shared pre-commit config](../shared/code-quality.md#pre-commit). For poetry-based projects, replace `uv run` with `poetry run`:

```bash
# Install (with uv)
uv run pre-commit install

# Install (with poetry)
poetry run pre-commit install
```

## Dependency Pinning

See [../shared/dependencies.md](../shared/dependencies.md#version-pinning-strategy) for the full pinning strategy. Summary: use minimum versions in `pyproject.toml`, let the lockfile handle exact pins.

## Type Annotations

Use type annotations everywhere. They're documentation that the type checker validates.

### ❌ Bad: no annotations

```python
def get_data(sources, api_client):
    results = {}
    for source in sources:
        data = api_client.fetch(source)
        results[source] = data
    return results
```

### ✅ Good: fully annotated

```python
def get_data(
    sources: list[DataSource],
    api_client: ApiClient,
) -> dict[DataSource, LoadedDataSource]:
    results: dict[DataSource, LoadedDataSource] = {}
    for source in sources:
        data = api_client.fetch(source)
        results[source] = LoadedDataSource(data_source=source, data=data)
    return results
```

## Minimum Python Version

Target Python 3.12+. Consider 3.13 for new projects. It gives you:

- `match` / `case` (3.10+)
- `StrEnum` (3.11+)
- `type` statement (3.12+)
- `list[str]` instead of `List[str]` (3.9+)
- Better error messages
- Faster startup

Enforce it:

```toml
[project]
requires-python = ">=3.12"
```

## CI Integration

Use the [shared lint-and-test workflow](../shared/ci-cd.md) as a base, then add pipeline-specific tools:

```yaml
jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync --frozen
      - run: uv run ruff check .
      - run: uv run ruff format --check .
      - run: uv run pyright
      - run: uv run deptry .
      - run: uv run vulture src/ --min-confidence 80

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync --frozen
      - run: uv run pytest tests/ -v --cov=my_pipeline
```
