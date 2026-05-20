# Data Analysis Best Practices

Best practices for building **reproducible, well-structured data analysis notebooks** in Python. These guidelines target analysis and calibration work in Jupyter — geospatial analysis, statistical calibration, impact assessment — where the deliverable is insight (plots, reports, calibrated parameters), not a deployed service.

## Guide Structure

| File | Topic |
|-------|---------------|
| [Project Structure](01-project-structure.md) | Directory layout, `notebooks/` vs `src/`, data folders |
| [Notebook Structure](02-notebook-structure.md) | Sections, cell ordering, markdown narrative, cell size |
| [Environment and Dependencies](03-environment-and-dependencies.md) | conda/mamba, `environment.yml`, pinning, kernel setup |
| [Configuration and Reproducibility](04-configuration-and-reproducibility.md) | Config cells, paths, parameters, system detection, validation |
| [Data Loading and I/O](05-data-loading-and-io.md) | Lazy loading, caching, file formats (NetCDF, Parquet, GeoJSON) |
| [Data Processing and Memory](06-data-processing-and-memory.md) | Chunked processing, xarray/dask, intermediate saves, large datasets |
| [Visualization](07-visualization.md) | matplotlib, folium, publication-quality plots, interactive maps |
| [Code Reuse and Packaging](08-code-reuse-and-packaging.md) | Extracting functions to `src/`, editable installs, notebook imports |
| [Documentation and Communication](09-documentation-and-communication.md) | Markdown cells, section summaries, stakeholder outputs |
| [Code Quality and Testing](10-code-quality-and-testing.md) | Linting notebooks, CI for notebooks, `nbstripout`, pre-commit |
