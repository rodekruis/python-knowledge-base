# 08 — Code Reuse and Packaging

## The Notebook → src/ Migration Pattern

Functions start their life in notebooks. When a function is used in **two or more notebooks**, it graduates to `src/`. This is the natural lifecycle of analytical code.

```
1. Write function inline in notebook (exploration)
2. Use it in a second notebook → time to extract
3. Move to src/mypackage/appropriate_module.py
4. Import in notebooks: from mypackage.module import function
```

### When to extract vs. when to keep inline

| Criterion | Keep inline (Section 0.5) | Extract to `src/` |
|-----------|--------------------------|-------------------|
| Used in how many notebooks? | 1 | 2+ |
| Lines of code? | <20 | >20 |
| Domain-specific logic? | No (glue code) | Yes (reusable logic) |
| Needs unit tests? | No | Yes |
| Likely to change often? | Yes (still exploring) | No (stable interface) |

## Structure src/ by Domain

Organize your package by what the code does, not by what notebook uses it.

### ✅ Good: domain-based structure

```
src/mypackage/
├── __init__.py
├── adapters/          # Data source adapters (GRIB, API, file download)
│   ├── __init__.py
│   ├── source_a.py
│   └── source_b.py
├── domain/            # Core dataclasses, config schemas, enums
│   ├── __init__.py
│   ├── config.py
│   └── models.py
├── geo/               # Geospatial operations (regrid, clip, project)
│   ├── __init__.py
│   ├── hydrobasins.py
│   └── raster_ops.py
├── models/            # Statistical models (EVT, impact functions)
│   ├── __init__.py
│   └── evt.py
├── ops/               # Orchestration (batch run, pipeline steps)
│   ├── __init__.py
│   └── pipeline.py
├── qc/                # Quality control checks
│   ├── __init__.py
│   └── validators.py
├── utils/             # Generic helpers (timing, memory, logging)
│   ├── __init__.py
│   ├── memory.py
│   └── paths.py
└── cli.py             # Command-line interface (typer/click)
```

### ❌ Bad: flat structure or by-notebook organization

```
src/
├── notebook_01_helpers.py   # Tied to a notebook — will rot
├── notebook_02_helpers.py   # Same problem
├── utils.py                 # 2000-line grab bag
└── everything_else.py       # Meaningless name
```

## Editable Install with pyproject.toml

Use `pip install -e .` to make your `src/` package importable from notebooks without path hacks. This is the modern Python packaging standard.

### pyproject.toml

```toml
[project]
name = "mypackage"
version = "0.1.0"
description = "Short description of the project"
requires-python = ">=3.12"
dependencies = [
    "numpy>=1.24",
    "xarray>=2023.1",
    "geopandas>=0.13",
    "rasterio>=1.3",
]

[project.optional-dependencies]
dev = ["pytest", "ruff", "nbqa", "pre-commit"]

[project.scripts]
mypackage = "mypackage.cli:app"

[tool.setuptools.packages.find]
where = ["src"]

[tool.ruff]
line-length = 100
select = ["E", "F", "I", "W"]

[tool.pytest.ini_options]
testpaths = ["tests"]
```

### Install in development mode

```bash
# From repo root:
pip install -e .

# Now this works from any notebook:
from mypackage.geo.hydrobasins import read_vector
from mypackage.models.evt import fit_gpd_model
```

## Path Setup for Notebooks (When Editable Install Isn't Used)

Sometimes you can't do an editable install (CI environments, quick experiments). Use a sys.path insert in Section 0:

### ✅ Good: explicit path setup in Section 0

```python
# === Section 0: Environment Setup ===
import sys
from pathlib import Path

REPO_ROOT = Path.cwd().parent.parent  # notebooks/ → repo root
SRC_PATH = REPO_ROOT / "src"

if str(SRC_PATH) not in sys.path:
    sys.path.insert(0, str(SRC_PATH))

# Now imports work
from mypackage.geo.hydrobasins import read_vector
from mypackage.domain.config import BasinConfig
```

### ❌ Bad: path hack in every cell that needs an import

```python
# Cell 5
sys.path.append("../../src")  # Relative, fragile, duplicated
from mypackage.geo import something

# Cell 12
sys.path.append("../../src")  # Again! Already in path!
from mypackage.models import something_else
```

## Section 0.5: Notebook-Local Helpers

For functions used only within one notebook, define them in Section 0.5. This keeps them visible and editable without the overhead of packaging.

### ✅ Good: Section 0.5 with focused helpers

```python
# === Section 0.5: Helper Functions (this notebook only) ===

def detect_calibration_mode(calibration_root: Path) -> str:
    """Detect whether this is a fresh run or a pre-existing calibration."""
    config_path = calibration_root / "run_config.json"
    if config_path.exists():
        config = json.loads(config_path.read_text())
        return config.get("selection_mode", "auto")
    return "fresh"


def get_flood_zone_boundary(muni_gdf: gpd.GeoDataFrame, basin_id: str) -> gpd.GeoDataFrame:
    """Extract the flood zone boundary for a given basin."""
    mask = muni_gdf['basin_id'] == basin_id
    if mask.sum() == 0:
        raise ValueError(f"No boundary found for basin: {basin_id}")
    return muni_gdf[mask].dissolve()
```

### ❌ Bad: helper functions scattered throughout the notebook

```python
# Cell 15 — deep in the analysis
def helper_a():  # reader didn't know this existed
    ...

# Cell 32 — even deeper
def helper_b():  # uses helper_a but it was defined 17 cells ago
    ...
```

## Template Pattern for New Data Sources

When your pipeline supports multiple data sources (APIs, rasters, local gauges), create adapter templates:

```python
# src/mypackage/adapters/base.py
from abc import ABC, abstractmethod
from pathlib import Path
import xarray as xr


class DataAdapter(ABC):
    """Base class for data source adapters."""

    @abstractmethod
    def download(self, target_dir: Path) -> list[Path]:
        """Download raw data files."""
        ...

    @abstractmethod
    def load(self, file_path: Path, chunks: str = 'auto') -> xr.Dataset:
        """Load data into xarray Dataset."""
        ...

    @abstractmethod
    def validate(self, ds: xr.Dataset) -> bool:
        """Validate loaded data against expected schema."""
        ...
```

```python
# src/mypackage/adapters/raster_source.py
class RasterSourceAdapter(DataAdapter):
    """Adapter for downloading raster tiles from a web API."""

    BASE_URL = "https://data.example.org/tiles/..."

    def download(self, target_dir: Path) -> list[Path]:
        tiles = self._get_tile_urls()
        paths = []
        for url in tiles:
            local = target_dir / url.split('/')[-1]
            if local.exists() and local.stat().st_size > 0:
                continue
            download_with_cache(url, local)
            paths.append(local)
        return paths
```

## When to Create a CLI

If your analysis can be parameterized and run non-interactively (e.g., "run for all basins"), add a CLI entry point:

```python
# src/mypackage/cli.py
import typer
from pathlib import Path

app = typer.Typer()

@app.command()
def run_analysis(
    region_id: str = typer.Argument(..., help="Region identifier"),
    config_path: Path = typer.Option("configs/default.yaml", help="Config file"),
    output_dir: Path = typer.Option("data/processed", help="Output directory"),
):
    """Run the analysis workflow for a single region."""
    from mypackage.ops.pipeline import AnalysisPipeline

    pipeline = AnalysisPipeline(region_id, config_path, output_dir)
    pipeline.run()

if __name__ == "__main__":
    app()
```

Usage:
```bash
mypackage run-analysis region_01 --config configs/regions/region_01.yaml
```