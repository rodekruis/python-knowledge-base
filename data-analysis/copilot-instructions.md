# Copilot Instructions — Data Analysis (Jupyter Notebooks)

Build reproducible Jupyter notebook workflows for analysis, statistical calibration, and geospatial modeling
at NLRC 510. Follow these conventions exactly. They are the consolidated 510 best practices; deviate only
with a documented reason.

## Working with the Analyst (Responsible AI Use)

Follow these principles: you assist, the human decides, reviews, and owns the output.

- **Think with the analyst, not for them.** Before generating non-trivial code, briefly discuss the
  approach and trade-offs. Explain statistical and geospatial choices so the analyst retains the reasoning.
- **Ask clarifying questions.** If a request is ambiguous, under-specified, or looks like the wrong
  approach, ask first — do not guess and produce a large answer. A short question beats a long wrong
  implementation.
- **Protect the analyst's understanding.** Don't just hand over answers the analyst can reason through.
  Prompt them to attempt it, explain *why* a method works, and call out assumptions and edge cases.
  Optimize for long-term comprehension over speed.
- **Don't reinvent the wheel.** Lean on `pandas`/`xarray`/`geopandas`/`scikit-learn` vectorized operations
  and existing `src/` utilities; don't hand-write what these libraries already do, and say why a library
  is better.
- **Small, reviewable changes.** Keep cells and diffs focused; separate refactors from analysis. Never
  produce massive, opaque changes.
- **Be transparent.** AI-assisted contributions should be disclosed (e.g. commit trailers) and must be
  fully understood by the human who commits them.
- **No secrets or PII into prompts/tools.** Never paste beneficiary, donor, or personnel data into AI
  tooling. Treat all generated code as untrusted until reviewed.

## Stack & Tooling

> **Deviation from the 510 default:** these projects use **conda/mamba**, NOT uv — analysis notebooks
> depend on compiled geospatial libraries (GDAL, NetCDF4, GEOS, PROJ) that install reliably via
> `conda-forge`. (Web/ETL projects still use uv.)

- **Python 3.11+** via **mamba** (`environment.yml`); register a named kernel per project.
- `pip install -e .` your `src/` package for live imports.
- Lint `src/` with **ruff**; lint notebook cells with **nbqa**; strip outputs with **nbstripout**.
- Test `src/` with **pytest**; smoke-test notebooks with **papermill**.

```bash
mamba env create -f environment.yml
mamba activate my-analysis
python -m ipykernel install --user --name my-analysis --display-name "Python (my-analysis)"
pip install -e .
nbstripout --install
```

## Project Structure

```
my-analysis/
├── notebooks/           # numbered, sequential: 01_*.ipynb → 02_*.ipynb → 03_*.ipynb
├── src/mypackage/       # importable reusable code, organized by DOMAIN not by notebook
│   ├── adapters/ domain/ geo/ models/ qc/ utils/ cli.py
├── scripts/             # CLI batch scripts
├── data/{raw,interim,processed}/   # NEVER committed (.gitkeep only)
├── docs/{getting-started,user-guides,technical}/
├── tests/               # pytest for src/
├── environment.yml      # conda env (minimum-version pins)
├── pyproject.toml       # package metadata + ruff/pytest config
└── README.md            # what it does, data needed, setup, how to run
```

- **Numbered notebooks tell a story** in execution order. Cross-notebook state passes **via files only**,
  never shared kernel state.
- Extract code to `src/` once it is used in **2+ notebooks** or exceeds ~20 lines; keep single-use glue in a
  notebook "Section 0.5: Helper Functions".

## Notebook Structure

Follow this section order with `##` markdown headers (enables ToC navigation):

1. **Section 0 — Environment Setup**: all imports here (never scattered), `sys.path`/logging setup.
2. **Section 0.5 — Helper Functions** (notebook-local only).
3. **Section 1 — Configuration** (see below).
4. **Sections 2–N — Analysis Steps**: load → process → model → output.
5. **Summary & Next Steps**: list outputs and link to the next notebook.
6. **Visualization**.

- **Keep cells under ~50 lines**, one responsibility each (helper defs may be longer).
- Open each section with a banner print and close with a status print (`✓ Section N complete: ...`).
- Every section gets a markdown cell explaining **WHAT** and **WHY** before the code.

## Configuration & Reproducibility

- Put a **configuration cell** right after imports: `UPPER_CASE` user params / algorithm flags, `lower_case`
  derived values, all paths via **`pathlib`** (never string paths). Auto-detect system RAM to set chunk sizes.
- **Validate config** before heavy work: assert required vars exist and input files are present, with
  actionable error messages (e.g. "➡️ Run Notebook 01 first").
- **Save a `run_config.json`** alongside outputs (params, algorithm options, timestamp, notebook name).
- Set random seeds for any stochastic step.

## Data Loading & I/O

- Match tool to format: **xarray** (NetCDF/GRIB), **rasterio** (GeoTIFF), **geopandas** (vector),
  **pandas/parquet** (tabular), **h5py** (HDF5).
- **Lazy-load** large arrays: `xr.open_dataset(path, chunks="auto")`. Validate after load (sizes, vars, CRS,
  % valid).
- Use the **"if exists, reload from disk; else compute and checkpoint"** pattern for any expensive step.
- Download with caching (skip if already present). Always `mkdir(parents=True, exist_ok=True)` before writing.
- Preferred formats: NetCDF (multidim), Parquet/GeoParquet (tabular/large vector), GeoJSON (small vector),
  HDF5 (domain objects), YAML/JSON (config).

## Data Processing & Memory

- **Keep arrays lazy until you need a scalar.** Call `.compute()` only on reductions (one number / small
  Series) — never `.compute()` a full array into RAM.
- Checkpoint to disk after operations >30s. Downcast to **float32** where precision allows. Validate dtypes
  and ranges (pre-flight asserts) before expensive ops.
- Free memory after heavy steps (`ds.close()`, `del`, `gc.collect()`) or use scoped `with xr.open_dataset(...)`.
- Provide a batch attempt with a **per-event fallback** on `MemoryError`/`ValueError`. Time phases with a
  `Timer` context manager.

## Visualization

- Set publication-quality `plt.rcParams` once at the start of the viz section (`savefig.dpi=300`,
  `bbox='tight'`). **One plot per cell.** Always save figures to a `figures/` dir at 300 DPI, then `show()`.
- Use **Cartopy** for projected maps, **Gridspec** for multi-panel, **Folium** for interactive maps.
  Choose domain-appropriate colormaps (`YlOrRd` depth, `RdBu_r` anomaly, `viridis` probability).

## Documentation & Communication

- Each notebook starts with a **Prerequisites** cell and "What this does / does NOT do". Document assumptions
  explicitly. Use Mermaid diagrams for workflows. Print progress for long loops.
- Use a consistent status-emoji convention (✅ done, ❌ error, ⚠️ warning, ⏳ in progress, ➡️ next step).
- Separate user guides from technical docs under `docs/`.

## Code Quality & Testing

- **Never commit notebook outputs** — `nbstripout --install` enforces this (large, non-reproducible,
  merge-conflict and security risk).
- Lint `src/` with ruff and notebook cells with `nbqa ruff notebooks/`. Use **pre-commit** chaining
  nbstripout + ruff + nbqa-ruff + `check-added-large-files (--maxkb=500)`.
- Unit-test `src/` code with pytest (mirror module layout). Smoke-test notebooks end-to-end with
  **papermill** (parametrize a small test region), marked `@pytest.mark.slow`.
- Maintain a comprehensive `.gitignore` covering `data/`, `*.nc/*.hdf5/*.tif/*.grib/*.parquet`,
  `.ipynb_checkpoints/`, build/env artifacts; keep `!data/**/.gitkeep`.

## environment.yml & pyproject.toml essentials

```yaml
# environment.yml — pin MINIMUM versions, not exact; let conda solve transitive deps
name: my-analysis
channels: [conda-forge, defaults]
dependencies:
  - python>=3.11
  - numpy>=1.24
  - pandas>=2.0
  - xarray>=2024.1
  - dask>=2024.1
  - geopandas>=0.14
  - rasterio>=1.3
  - cartopy>=0.22
  - matplotlib>=3.8
  - jupyterlab>=4.0
  - ipykernel>=6.0
  - pytest>=8.0
  - ruff>=0.8
  - pip:
      - nbstripout>=0.7
      - papermill
      - nbqa
```

```toml
[tool.ruff]
line-length = 100
target-version = "py312"
[tool.ruff.lint]
select = ["E","F","I","W","UP","B","SIM"]

[tool.setuptools.packages.find]
where = ["src"]
```

Export an exact lockfile for operational/production runs: `mamba env export --no-builds > environment.lock.yml`.

## Anti-Patterns

Committing notebook outputs or `data/` · scattered imports · cells >50 lines doing many things ·
`.compute()` on full arrays · float64 by default for large rasters · string paths (use pathlib) ·
recomputing expensive steps instead of checkpointing · shared kernel state between notebooks ·
exact version pins in `environment.yml` · copy-pasting the same function across notebooks (extract to `src/`) ·
hand-writing what `pandas`/`xarray`/`geopandas`/`scikit-learn` already do.
