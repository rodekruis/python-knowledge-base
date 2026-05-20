# 05 — Data Loading and I/O

## Choose the Right Tool for Each Data Format

Every data format has a preferred library. Don't use pandas to read NetCDF. Don't use xarray to read shapefiles. Use the right tool:

| Format | Library | Use case |
|--------|---------|----------|
| NetCDF / GRIB | `xarray` | Multidimensional arrays (climate, flood maps) |
| GeoTIFF | `rasterio` | Raster data with CRS |
| Shapefile / GeoJSON | `geopandas` | Vector data (boundaries, points) |
| CSV / Parquet | `pandas` | Tabular data |
| HDF5 | `h5py` or domain library | CLIMADA objects, large arrays |

## Use xarray with Lazy Loading

For large NetCDF or GRIB files, **always** pass `chunks='auto'` to enable dask-backed lazy loading. This means the data isn't loaded into memory until you explicitly call `.compute()` or `.values`.

### ✅ Good: lazy loading with chunks

```python
ds = xr.open_dataset(RETURN_PERIOD_NC, chunks='auto')
print(f"✓ Loaded: {ds.sizes}")
# Data is NOT in RAM yet — only metadata is read
```

### ❌ Bad: loading entire file into memory immediately

```python
ds = xr.open_dataset(RETURN_PERIOD_NC)
# The entire multi-GB file is now in RAM
```

### When to use which chunk strategy

```python
# Auto chunks (good default for most files)
ds = xr.open_dataset(file, chunks='auto')

# Explicit chunks for spatial operations (when you know access patterns)
ds = xr.open_dataset(file, chunks={'latitude': 256, 'longitude': 256})

# No chunks for small files (<100 MB)
ds = xr.open_dataset(small_file)
```

## The "If Exists, Reload from Disk" Pattern

This is the single most important I/O pattern for analysis notebooks. Heavy processing steps (merging tiles, regridding) can take minutes to hours. Save intermediate results and reload them on re-run.

### ✅ Good: cache intermediate results

```python
flood_maps_path = INTERIM_DIR / "flood-maps_intermediate.nc"

if flood_maps_path.exists():
    print(f"✓ Loading cached flood maps: {flood_maps_path.name}")
    flood_maps = xr.open_dataset(flood_maps_path, chunks='auto')
else:
    print("⚠️ No cached file — computing from raw tiles...")
    flood_maps = merge_jrc_tiles(tile_dir, aoi_boundary)
    flood_maps.to_netcdf(flood_maps_path)
    print(f"✓ Saved intermediate: {flood_maps_path.name}")

print(f"  Shape: {flood_maps.sizes}")
```

### ❌ Bad: recomputing every time

```python
# This takes 15 minutes and runs every time you restart the kernel
flood_maps = merge_jrc_tiles(tile_dir, aoi_boundary)
```

### Pattern for multiple intermediate files

```python
INTERMEDIATE_FILES = {
    "flood_maps": INTERIM_DIR / "flood-maps_intermediate.nc",
    "rp_regrid": INTERIM_DIR / "return-period_regrid_all.nc",
    "hazard": OUTPUT_DIR / f"climada_hazard_{BASIN_ID}.hdf5",
}

def load_or_compute(key, compute_fn, **kwargs):
    """Load from cache if available, otherwise compute and save."""
    path = INTERMEDIATE_FILES[key]
    if path.exists():
        print(f"✓ Cache hit: {path.name}")
        return xr.open_dataset(path, chunks='auto')
    else:
        print(f"⏳ Computing: {key}...")
        result = compute_fn(**kwargs)
        result.to_netcdf(path)
        print(f"✓ Saved: {path.name}")
        return result
```

## Download Remote Data with Caching

When downloading files from HTTP, always check if the file already exists and has non-zero size before downloading again. This is polite to servers and avoids wasting bandwidth on re-runs.

### ✅ Good: download with local caching

```python
import requests
from pathlib import Path

def download_with_cache(url: str, local_path: Path, description: str = "") -> Path:
    """Download file from URL, skip if already cached locally."""
    if local_path.exists() and local_path.stat().st_size > 0:
        print(f"  ✓ Already cached: {local_path.name}")
        return local_path

    print(f"  ⬇ Downloading: {description or url}")
    local_path.parent.mkdir(parents=True, exist_ok=True)

    response = requests.get(url, stream=True, timeout=60)
    response.raise_for_status()

    with open(local_path, 'wb') as f:
        for chunk in response.iter_content(chunk_size=8192):
            f.write(chunk)

    print(f"  ✓ Saved: {local_path.name} ({local_path.stat().st_size / 1e6:.1f} MB)")
    return local_path
```

### ❌ Bad: re-downloading every time

```python
# Downloads 500 MB of tiles every single run
for tile_url in jrc_tile_urls:
    r = requests.get(tile_url)
    open(f"data/{tile_url.split('/')[-1]}", 'wb').write(r.content)
```

## Always Validate After Loading

Never trust that a loaded file contains what you expect. Print shape, dimensions, and check for obvious issues (all-NaN, wrong CRS, unexpected dtypes).

### ✅ Good: validate immediately after loading

```python
# Load and validate
ds_rp = xr.open_dataset(RETURN_PERIOD_NC, chunks='auto')

# Shape validation
print(f"✓ Loaded: {ds_rp.sizes}")
print(f"  Variables: {list(ds_rp.data_vars)}")
print(f"  CRS: {ds_rp.attrs.get('crs', 'NOT SET')}")

# Data quality check
for var in ds_rp.data_vars:
    n_valid = np.isfinite(ds_rp[var]).sum().compute()
    n_total = ds_rp[var].size
    pct_valid = 100 * int(n_valid) / n_total
    print(f"  {var}: {pct_valid:.1f}% valid ({int(n_valid):,} / {n_total:,})")
```

### ❌ Bad: no validation

```python
ds_rp = xr.open_dataset(RETURN_PERIOD_NC)
# Proceeds to use ds_rp without checking anything
# Fails 10 minutes later with an unhelpful error
```

## File Format Recommendations

Choose file formats based on the data type, not convenience:

| Data type | Recommended format | Why |
|-----------|-------------------|-----|
| Multidim arrays (flood maps, climate) | **NetCDF** | Self-describing, chunked, lazy loading, metadata |
| Tabular data (exposure, stations) | **Parquet** | Columnar, compressed, fast, typed |
| Small vector datasets | **GeoJSON** | Human-readable, git-friendly |
| Large vector datasets | **GeoParquet** | Fast, compressed, spatial indexing |
| CLIMADA objects | **HDF5** | Native format with `.from_hdf5()` |
| Configuration | **YAML** or **JSON** | Human-readable, version-controllable |

### ❌ Bad: CSV for everything

```python
# Writing a 500-column exposure table to CSV
df.to_csv("exposure.csv")  # Slow, large, no types, no compression
```

### ✅ Good: Parquet for tabular data

```python
df.to_parquet("exposure.parquet", compression="snappy")
# 10x smaller, 5x faster to read, preserves dtypes
```

## Use pathlib, Not String Paths

Always use `pathlib.Path` for file paths. It handles OS differences, provides `.exists()`, `.mkdir()`, and path concatenation with `/`.

### ✅ Good: pathlib

```python
from pathlib import Path

REPO_ROOT = Path.cwd().parent.parent
DATA_DIR = REPO_ROOT / "data" / "processed" / BASIN_ID
output_path = DATA_DIR / f"flood-maps_{BASIN_ID}.nc"

output_path.parent.mkdir(parents=True, exist_ok=True)
```

### ❌ Bad: string concatenation

```python
import os
output_path = os.path.join("data", "processed", basin_id, "flood-maps_" + basin_id + ".nc")
os.makedirs(os.path.dirname(output_path), exist_ok=True)
```