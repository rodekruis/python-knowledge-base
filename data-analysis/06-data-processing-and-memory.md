# 06 — Data Processing and Memory Management

## Keep Arrays Lazy Until You Need a Number

The most common memory mistake in geospatial notebooks is calling `.values` or `.compute()` too early. With xarray + dask, arrays remain lazy (just metadata) until you force computation. **Only compute statistics, never full arrays unless writing to disk.**

### ✅ Good: lazy until the last moment

```python
# These are all lazy — no RAM used
rp_stack = xr.concat(rp_arrays, dim='return_period')
rp_stack_f32 = rp_stack.astype(np.float32)
flood_masked = rp_stack_f32.where(rp_stack_f32 > 0.01)

# Only this forces computation (one number, not the whole array)
total_flooded = (flood_masked > 0).sum().compute()
print(f"✓ Flooded cells: {int(total_flooded):,}")
```

### ❌ Bad: forcing computation on the whole array

```python
# Loads entire multi-GB array into RAM
rp_stack = xr.concat(rp_arrays, dim='return_period')
rp_values = rp_stack.values  # 💥 OOM on 8 GB machine

# Or equivalently:
rp_computed = rp_stack.compute()  # Same explosion
```

### The rule: `.compute()` only for scalars and small summaries

```python
# ✅ OK — returns one number
mean_depth = flood_maps.mean().compute()

# ✅ OK — returns one small Series
stats_per_rp = flood_maps.mean(dim=['latitude', 'longitude']).compute()

# ❌ NEVER — returns the full array
all_data = flood_maps.compute()
```

## Save Intermediate Results Between Heavy Steps

When a processing step takes more than 30 seconds, save the result to disk. This avoids recomputing on kernel restart and lets you checkpoint long workflows.

### ✅ Good: checkpoint after expensive operations

```python
# Step 1: Merge tiles (expensive: 5-10 minutes)
flood_maps_path = INTERIM_DIR / "flood-maps_intermediate.nc"

if flood_maps_path.exists():
    flood_maps = xr.open_dataset(flood_maps_path, chunks='auto')
    print(f"✓ Loaded from cache: {flood_maps_path.name}")
else:
    flood_maps = merge_and_clip_tiles(tile_paths, aoi_boundary)
    flood_maps.to_netcdf(flood_maps_path)
    print(f"✓ Saved checkpoint: {flood_maps_path.name}")
```

### ❌ Bad: 45-minute pipeline with no checkpoints

```python
# If cell 8 fails, you have to re-run everything from cell 1
tiles = download_tiles()          # 10 min
merged = merge_tiles(tiles)       # 15 min
regridded = regrid(merged)        # 10 min
masked = apply_mask(regridded)    # 5 min
hazard = create_hazard(masked)    # 5 min - crashes on a dtype mismatch
# Now you wait 40 minutes again
```

## Use float32 Over float64 When Precision Allows

For flood depths, return periods, and most physical quantities, 32-bit precision is more than enough. This halves memory usage.

### ✅ Good: downcast after concatenation

```python
# Concat as lazy arrays, THEN cast (keeps lazy, doesn't allocate)
rp_stack = xr.concat(rp_arrays, dim='return_period')
rp_stack = rp_stack.astype(np.float32)
print(f"✓ Stack dtype: {rp_stack.dtype} (memory halved)")
```

### ❌ Bad: defaulting to float64 everywhere

```python
# numpy defaults to float64 — 2x the memory for no benefit
depths = np.zeros((10000, 10000))  # 800 MB
# Should be:
depths = np.zeros((10000, 10000), dtype=np.float32)  # 400 MB
```

### When to keep float64

- Financial calculations (currency precision matters)
- Coordinate transformations (reprojections, CRS operations)
- Cumulative statistics over very large datasets (numerical stability)
- When a downstream library requires it (validate first!)

## Validate Dtypes Before Expensive Operations

Some libraries (CLIMADA, scipy) are strict about input dtypes. Check before feeding data into expensive computations — failing after 10 minutes of processing is worse than failing before.

### ✅ Good: validate dtype before CLIMADA

```python
# CLIMADA expects float64 intensity
assert flood_maps_sel.dtype == np.float64, (
    f"Expected float64, got {flood_maps_sel.dtype}. "
    f"Cast with: flood_maps_sel.astype(np.float64)"
)

# Now safe to pass to Hazard.set_raster()
hazard = Hazard.set_raster(
    [flood_maps_sel[i].values for i in range(len(return_periods))],
    centroids=centroids,
)
```

### ❌ Bad: no validation

```python
# Passes float32 to a function that silently produces wrong results with float32
hazard = Hazard.set_raster(flood_maps_sel.values, ...)
# Results are subtly wrong — you won't notice until validation
```

## Phase Timing with Context Manager

For long-running sections, measure and print elapsed time. This helps identify bottlenecks and gives the reader an expectation of runtime.

### ✅ Good: Timer context manager

```python
import time
from contextlib import contextmanager

@contextmanager
def Timer(label: str):
    """Time a code block and print elapsed time."""
    start = time.perf_counter()
    print(f"⏳ {label}...")
    yield
    elapsed = time.perf_counter() - start
    if elapsed < 60:
        print(f"✓ {label}: {elapsed:.1f}s")
    else:
        print(f"✓ {label}: {elapsed/60:.1f} min")

# Usage
with Timer("Regridding return periods"):
    rp_regrid = petals_regrid(rp_stack, flood_maps_sel, method=REGRID_METHOD)
```

Output:
```
⏳ Regridding return periods...
✓ Regridding return periods: 2.3 min
```

## Clean Up Memory After Heavy Steps

After a processing step, close datasets and delete large arrays you no longer need. Python's garbage collector doesn't always free memory immediately.

### ✅ Good: explicit cleanup

```python
# Done with raw tiles — close and delete
ds_raw.close()
del ds_raw, tile_arrays
gc.collect()
print(f"✓ Freed memory from raw tile data")
```

### Pattern: scoped loads

Only load what you need from a dataset, then close it:

```python
# Load only the variable you need
with xr.open_dataset(RETURN_PERIOD_NC, chunks='auto') as ds:
    rp_100 = ds['return_period_RP100'].load()  # Load just this one variable
# ds is now closed, file handle released
```

## Event-by-Event Fallback Pattern

When processing multiple events (return periods, timesteps), try batch processing first. If it fails (OOM, library bug), fall back to one-at-a-time processing.

### ✅ Good: batch with single-event fallback

```python
try:
    # Try multi-event (fast, but memory-intensive)
    print("⏳ Attempting multi-event hazard creation...")
    hazard = create_hazard_batch(flood_maps_all, return_periods)
    print(f"✓ Multi-event success: {hazard.size} events")
except (MemoryError, ValueError) as e:
    # Fall back to event-by-event (slower, but memory-safe)
    print(f"⚠️ Multi-event failed ({type(e).__name__}), falling back to per-event...")
    hazards = []
    for i, rp in enumerate(return_periods):
        print(f"  Processing RP {rp} ({i+1}/{len(return_periods)})...")
        h = create_hazard_single(flood_maps_all[i], rp)
        hazards.append(h)
    hazard = Hazard.concat(hazards)
    print(f"✓ Per-event fallback complete: {hazard.size} events")
```

### ❌ Bad: no fallback, just crash

```python
# If this OOMs, the user has to manually figure out what to do
hazard = create_hazard_batch(flood_maps_all, return_periods)
```

## Chunked Processing for Spatial Operations

When processing rasters that don't fit in memory, process them in spatial chunks.

```python
def process_in_chunks(ds, func, chunk_size=512):
    """Process a dataset in spatial chunks to avoid OOM."""
    ny, nx = ds.sizes['latitude'], ds.sizes['longitude']
    results = []

    total_chunks = ((ny // chunk_size) + 1) * ((nx // chunk_size) + 1)
    chunk_idx = 0

    for y_start in range(0, ny, chunk_size):
        for x_start in range(0, nx, chunk_size):
            chunk_idx += 1
            chunk = ds.isel(
                latitude=slice(y_start, y_start + chunk_size),
                longitude=slice(x_start, x_start + chunk_size),
            )
            result = func(chunk.compute())
            results.append(result)

            if chunk_idx % 10 == 0:
                print(f"  Processed {chunk_idx}/{total_chunks} chunks...")

    return xr.combine_by_coords(results)
```

## Validate Inputs Before Expensive Operations

Check shapes, ranges, and bounds before starting a computation that takes minutes.

### ✅ Good: pre-flight checks

```python
# Pre-flight validation
assert flood_maps.sizes['latitude'] > 0, "Empty latitude dimension"
assert flood_maps.sizes['longitude'] > 0, "Empty longitude dimension"
assert flood_maps.min().compute() >= 0, "Negative flood depths — check units"
assert flood_maps.max().compute() < 100, f"Max depth {float(flood_maps.max().compute())}m — implausible"

print(f"✓ Validation passed: {flood_maps.sizes}")
print(f"  Depth range: [{float(flood_maps.min().compute()):.2f}, {float(flood_maps.max().compute()):.2f}] m")
```