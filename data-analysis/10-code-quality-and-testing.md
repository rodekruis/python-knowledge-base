# 10 — Code Quality and Testing

> **Shared foundations**: See [../shared/code-quality.md](../shared/code-quality.md) for the canonical ruff config, type checking, and pre-commit setup. This guide covers **notebook-specific** tooling (nbstripout, nbqa, papermill) that extends the shared baseline.

## nbstripout: Keep Notebook Diffs Clean

Notebook outputs (plots, dataframes, cell execution counts) are binary blobs that pollute git diffs. Use `nbstripout` to automatically strip them before commit.

### Setup

```bash
pip install nbstripout
nbstripout --install  # Installs as a git filter for .ipynb files
```

This means:
- Outputs are stripped on `git add` (transparent to the user)
- The working copy still has outputs (you see them in Jupyter)
- `git diff` only shows actual code changes

### ❌ Bad: committing notebooks with outputs

```diff
# git diff shows 500 lines of base64-encoded PNG data
+ "image/png": "iVBORw0KGgoAAAANSUhEUgAAA..."  # 🚫
```

### ✅ Good: nbstripout removes outputs before commit

```diff
# git diff shows only code changes
- threshold = 0.95
+ threshold = 0.90
```

## ruff for src/ Code

Use `ruff` for linting and formatting the `src/` package. It's fast, comprehensive, and covers what flake8 + isort + pycodestyle used to do.

### pyproject.toml configuration

```toml
[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "F",    # pyflakes
    "I",    # isort (import sorting)
    "W",    # pycodestyle warnings
    "UP",   # pyupgrade (modern Python syntax)
    "B",    # flake8-bugbear (common bugs)
    "SIM",  # flake8-simplify
]
ignore = ["E501"]  # Line length handled by formatter

[tool.ruff.format]
quote-style = "double"
```

### Run manually

```bash
ruff check src/            # Lint
ruff format src/           # Format
ruff check src/ --fix      # Auto-fix what's possible
```

## nbqa: Run Linters on Notebook Code Cells

`nbqa` lets you run ruff, black, or mypy on the code cells of Jupyter notebooks — without touching markdown.

### Usage

```bash
pip install nbqa

# Lint notebook code cells
nbqa ruff notebooks/

# Format notebook code cells
nbqa ruff notebooks/ --fix

# Check imports
nbqa isort notebooks/ --check
```

### ✅ Good: notebooks pass the same lint rules as src/

```bash
$ nbqa ruff notebooks/02_HazardModeling.ipynb
All checks passed!
```

### ❌ Bad: src/ is clean but notebooks have style violations

```bash
# src/ passes ruff but notebooks are inconsistent
$ ruff check src/  # ✅ clean
$ nbqa ruff notebooks/  # ❌ 47 violations
```

## Pre-commit Hooks: Automated Quality Gates

Use pre-commit to run quality checks automatically before every commit. This catches mistakes before they reach the remote.

### .pre-commit-config.yaml

```yaml
repos:
  # Strip notebook outputs
  - repo: https://github.com/kynan/nbstripout
    rev: 0.7.1
    hooks:
      - id: nbstripout

  # Lint and format src/ code
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  # Lint notebook code cells
  - repo: https://github.com/nbQA-dev/nbQA
    rev: 1.9.1
    hooks:
      - id: nbqa-ruff
        args: [--fix]

  # Prevent large files from being committed
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: check-added-large-files
        args: ['--maxkb=500']
      - id: end-of-file-fixer
      - id: trailing-whitespace
```

### Install

```bash
pip install pre-commit
pre-commit install
```

Now every `git commit` automatically:
1. Strips notebook outputs
2. Lints and formats src/ code
3. Lints notebook code cells
4. Blocks accidental large file commits

## Test src/ Code with pytest

Functions in `src/` should have unit tests. This catches regressions when you refactor and gives confidence that extracted code works correctly.

### Test structure

```
tests/
├── conftest.py            # Shared fixtures (sample data, paths)
├── test_geo/
│   ├── test_hydrobasins.py
│   └── test_raster_ops.py
├── test_models/
│   └── test_evt.py
└── test_adapters/
    └── test_jrc_flood.py
```

### Example test

```python
# tests/test_geo/test_raster_ops.py
import numpy as np
import xarray as xr
import pytest
from mypackage.geo.raster_ops import regrid_to_target


@pytest.fixture
def sample_grid():
    """Create a small test grid."""
    return xr.DataArray(
        np.random.rand(10, 10),
        dims=['latitude', 'longitude'],
        coords={
            'latitude': np.linspace(14.0, 15.0, 10),
            'longitude': np.linspace(121.0, 122.0, 10),
        }
    )


def test_regrid_preserves_extent(sample_grid):
    """Regridded output should cover the same geographic extent."""
    target = xr.DataArray(
        np.zeros((20, 20)),
        dims=['latitude', 'longitude'],
        coords={
            'latitude': np.linspace(14.0, 15.0, 20),
            'longitude': np.linspace(121.0, 122.0, 20),
        }
    )
    result = regrid_to_target(sample_grid, target, method='bilinear')

    assert result.latitude.min() == pytest.approx(14.0, abs=0.1)
    assert result.latitude.max() == pytest.approx(15.0, abs=0.1)
    assert result.shape == (20, 20)


def test_regrid_rejects_mismatched_crs(sample_grid):
    """Should raise ValueError if CRS doesn't match."""
    target_wrong_crs = sample_grid.copy()
    target_wrong_crs.attrs['crs'] = 'EPSG:32651'
    sample_grid.attrs['crs'] = 'EPSG:4326'

    with pytest.raises(ValueError, match="CRS mismatch"):
        regrid_to_target(sample_grid, target_wrong_crs)
```

### Run tests

```bash
pytest tests/ -v --tb=short
```

## Papermill for Automated Notebook Execution

Use `papermill` to run notebooks programmatically in CI. This proves notebooks actually execute top-to-bottom without error.

### Parameterized execution

```bash
# Run notebook with specific parameters
papermill notebooks/02_HazardModeling.ipynb \
    outputs/02_executed.ipynb \
    -p BASIN_ID "region_01" \
    -p RUN_TAG "ci-test" \
    -p AUTO_DETECT false
```

### CI smoke test

```python
# tests/test_notebooks.py
import papermill as pm
import pytest
from pathlib import Path

NOTEBOOKS_DIR = Path("notebooks")
OUTPUT_DIR = Path("test_outputs")


@pytest.mark.slow
@pytest.mark.parametrize("notebook", [
    "01_Calibration.ipynb",
    "02_HazardModeling.ipynb",
])
def test_notebook_runs_without_error(notebook, tmp_path):
    """Smoke test: notebook executes top-to-bottom."""
    pm.execute_notebook(
        str(NOTEBOOKS_DIR / notebook),
        str(tmp_path / notebook),
        parameters={"BASIN_ID": "test_basin", "RUN_TAG": "ci-smoke"},
        kernel_name="python3",
    )
```

## CI Workflow

A complete CI pipeline for analysis repos:

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: |
          pip install -e ".[dev]"

      - name: Lint src/
        run: ruff check src/

      - name: Format check
        run: ruff format --check src/

      - name: Lint notebooks
        run: nbqa ruff notebooks/

      - name: Run unit tests
        run: pytest tests/ -v --ignore=tests/test_notebooks.py

      # Optional: run notebooks (slow, needs data)
      # - name: Smoke test notebooks
      #   run: pytest tests/test_notebooks.py -v --timeout=600
```

## .gitignore for Analysis Repos

### ✅ Good: comprehensive .gitignore

```gitignore
# Data files (download via script, don't commit)
data/
*.nc
*.hdf5
*.tif
*.grib
*.grib2
*.parquet

# Notebook outputs (stripped by nbstripout, but just in case)
notebooks/.ipynb_checkpoints/

# Python
__pycache__/
*.pyc
*.egg-info/
.eggs/
dist/
build/

# Environment
.venv/
*.conda/

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db

# Keep directory structure
!data/.gitkeep
!data/raw/.gitkeep
!data/interim/.gitkeep
!data/processed/.gitkeep
```

### ❌ Bad: missing critical ignores

```gitignore
# Only ignores .pyc — forgets data, outputs, checkpoints
*.pyc
```

## Never Commit Notebook Outputs

This bears repeating: **never commit notebook outputs to git**. They are:
- Large (plots as base64 PNG)
- Non-reproducible (different on each run)
- Merge-conflict magnets (binary data in JSON)
- Security risks (may contain API keys in output)

Use nbstripout (automated) or "Clear All Outputs" before committing (manual, error-prone).