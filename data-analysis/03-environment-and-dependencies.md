# 03 — Environment and Dependencies

## Use conda/mamba, Not pip

Analysis notebooks depend on compiled C/Fortran libraries (GDAL, NetCDF4, GEOS, PROJ). These are a nightmare to install with pip but work out of the box with conda.

**Use [mamba](https://mamba.readthedocs.io/) instead of conda** — it's the same thing but 10-100x faster at solving environments.

```bash
# Install mamba (if not already present)
conda install -n base -c conda-forge mamba

# Create environment from definition
mamba env create -f environment.yml

# Activate
mamba activate my-analysis
```

## Define the Environment in `environment.yml`

Every analysis repo must have an `environment.yml` at root. This is the single source of truth for the environment.

```yaml
# environment.yml
name: my-analysis
channels:
  - conda-forge
  - defaults
dependencies:
  # Core scientific stack
  - python>=3.11
  - numpy>=1.24
  - pandas>=2.0
  - xarray>=2024.1
  - dask>=2024.1
  - scipy>=1.12

  # Geospatial
  - geopandas>=0.14
  - rasterio>=1.3
  - shapely>=2.0
  - pyproj>=3.6
  - cartopy>=0.22
  - folium>=0.15

  # Visualization
  - matplotlib>=3.8
  - seaborn>=0.13

  # Domain-specific
  - climada>=3.0
  - climada-petals>=1.0
  - pyextremes>=2.3

  # Jupyter
  - jupyterlab>=4.0
  - ipywidgets>=8.0
  - ipykernel>=6.0

  # Development
  - pytest>=8.0
  - ruff>=0.8

  # Pip-only packages (last resort)
  - pip:
    - nbstripout>=0.7
```

## Key Rules

### 1. Pin minimum versions, not exact

```yaml
# ✅ Good: allows compatible updates
- numpy>=1.24
- pandas>=2.0

# ❌ Bad: locks everything, causes conflicts
- numpy==1.24.3
- pandas==2.0.1
```

### 2. Don't manually install transitive dependencies

The reference repo explicitly warns: *"Do NOT manually install numpy, pandas, xarray, scipy, geopandas, or rasterio — these are automatically included by the 4 core packages."*

If package A depends on package B, only list A. Installing B separately risks version conflicts.

### 3. Use lockfiles for production

For operational pipelines deployed from notebooks, export an exact lockfile:

```bash
# Export exact environment (for reproduction)
mamba env export --no-builds > environment.lock.yml

# Recreate exact environment
mamba env create -f environment.lock.yml
```

### 4. Separate analysis and operational environments

```
environment.yml       # Full analysis env (Jupyter, plotting, dev tools)
requirements-ops.txt  # Minimal production deps (for scheduled runs)
```

The reference repo does this with `requirements.txt` (core) and `requirements-dev.txt` (development).

## Register the Kernel

After creating the environment, register it as a Jupyter kernel:

```bash
mamba activate my-analysis
python -m ipykernel install --user --name my-analysis --display-name "Python (my-analysis)"
```

This ensures the notebook uses the correct environment, even when launched from a different one.

## Editable Package Install

If you have a `src/` package (see [08 — Code Reuse](08-code-reuse-and-packaging.md)), install it in development mode:

```bash
mamba activate my-analysis
pip install -e .
```

This lets you `from mypackage import ...` in notebooks while editing the source files live.