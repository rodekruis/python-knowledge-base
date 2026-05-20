# 01 — Project Structure

## Standard Layout for Analysis Repos

```
my-analysis/
├── notebooks/               # Jupyter notebooks (numbered, sequential)
│   ├── 01_Calibration.ipynb
│   ├── 02_HazardWorkflow.ipynb
│   └── 03_ImpactModeling.ipynb
├── src/mypackage/           # Reusable Python code (importable)
│   ├── __init__.py
│   ├── adapters/            # Data source adapters (GRIB, API, file I/O)
│   ├── domain/              # Core logic, dataclasses, config schemas
│   ├── geo/                 # Geospatial operations
│   ├── models/              # Statistical models (EVT, impact, etc.)
│   ├── qc/                  # Quality control checks
│   └── utils/               # Helpers (memory, paths, plotting)
├── scripts/                 # CLI scripts for batch processing
├── data/                    # Data directory (NOT in git)
│   ├── raw/                 # Untouched source data
│   ├── interim/             # Intermediate outputs between notebooks
│   └── processed/           # Final analysis outputs
├── docs/                    # Documentation, user guides
├── tests/                   # Tests for src/ code
├── environment.yml          # Conda environment definition
├── pyproject.toml           # Package metadata + tool config
├── .gitignore
└── README.md
```

## Key Principles

### 1. Numbered notebooks tell a story

Notebooks should be numbered in execution order. A reader should be able to run `01 → 02 → 03` and reproduce the full analysis. If notebook B depends on output from notebook A, that dependency must be explicit (file path, not shared notebook state).

### 2. `data/` is never committed to git

```gitignore
# .gitignore
data/
*.nc
*.hdf5
*.tif
*.grib
*.parquet
!data/.gitkeep
```

Use `.gitkeep` files to preserve the directory structure in git. Document where to download data in the README.

### 3. `src/` for anything used more than once

If a function appears in two or more notebooks, move it to `src/`. This avoids copy-paste drift and makes the code testable.

```python
# In notebook:
from mypackage.geo.hydrobasins import read_vector, select_context_polygon
from mypackage.models.evt import fit_gpd_model
```

### 4. Separate configuration from analysis

Use YAML configs per region (`configs/regions/region_a.yaml`). This means the same notebook can process different regions without code changes.

### 5. README must answer four questions

Every analysis repo README must answer:

| Question | Section |
|----------|---------|
| What does this code do? | Introduction |
| What data does it need? | Input and output |
| How do I set it up? | Installation / Environment |
| How do I run it? | Step-by-step instructions |

The reference repo does this well with "What This Project Does", "Quick Navigation", "Installation", and "Troubleshooting" sections.

## Checklist

| Aspect | Target |
|--------|--------|
| Numbered notebooks | `01_*`, `02_*`, `03_*` — sequential execution order |
| `src/` package | `src/mypackage/` with adapters, domain, models, geo |
| `data/` directory | `data/raw/`, `data/interim/`, `data/processed/` — never committed |
| Config files | YAML per region/variant in `configs/` |
| Scripts for batch ops | `scripts/` directory for CLI workflows |
| Tests | Unit + integration with meaningful coverage |
| README | Answers: what, what data, how to set up, how to run |
| `.gitignore` | Ignores data, outputs, caches |
| Docs | `docs/` with user guides and architecture |
