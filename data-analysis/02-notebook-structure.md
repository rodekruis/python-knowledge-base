# 02 — Notebook Structure

## Every Notebook is a Numbered Sequence of Sections

A notebook should read like a technical report. Each section has a clear purpose, declared in a markdown cell, followed by code cells that implement it.

### The section pattern

```
┌─────────────────────────────────────────┐
│ ## Section N: Title                      │  ← Markdown: what and why
│ Brief description of purpose             │
└──────────────────────────────────────────┘
┌──────────────────────────────────────────┐
│ # Code cell                              │  ← Code: the how
│ result = do_the_thing()                  │
│ print(f"✓ Section N complete: {summary}")│  ← Status print at end
└──────────────────────────────────────────┘
```

### Standard section order

| Section | Purpose | Example |
|---------|---------|---------|
| 0 | **Environment Setup** | Imports, path setup, logging, workarounds |
| 0.5 | **Helper Functions** | Reusable functions defined for this notebook |
| 1 | **Configuration** | User parameters, system detection, path construction |
| 2–N | **Analysis Steps** | Data loading → processing → modeling → output |
| N+1 | **Summary & Next Steps** | Report what was produced, link to next notebook |
| N+2 | **Visualization** | Publication-quality plots, interactive maps |

The reference repo follows this exactly: Section 0 (env), 0.5 (helpers), 1 (config), 2 (load data), 3–5 (processing), 6 (create output), 7 (summary), 8 (visualization).

## Cell Size: Keep It Under 50 Lines

Long cells are hard to debug (which line failed?), hard to re-run partially, and hard to review. **Split aggressively.**

### ❌ Bad: 200-line mega-cell

```python
# This cell does imports, config, data loading, processing, and plotting
import numpy as np
# ... 195 more lines ...
plt.savefig("output.png")
```

### ✅ Good: one responsibility per cell

```python
# Cell 1: Load calibration data
ds = xr.open_dataset(RETURN_PERIOD_NC, chunks='auto')
print(f"✓ Loaded: {ds.sizes}")
```

```python
# Cell 2: Validate data quality
for var in ds.data_vars:
    valid = np.isfinite(ds[var]).sum().compute()
    print(f"  {var}: {int(valid):,} valid cells")
```

```python
# Cell 3: Regrid to target resolution
rp_regrid = petals_regrid(rp_stack, flood_maps_sel, method=REGRID_METHOD)
print(f"✓ Regridded: {rp_regrid.shape}")
```

**Exception**: Helper function definitions (Section 0.5) can be longer, because the function is a self-contained unit.

## Start Each Section with a Status Banner

Print a clear banner at the start and end of each section. This makes it trivial to scan the output and find where something went wrong.

```python
print("\n" + "=" * 70)
print("SECTION 4: Merge JRC Flood Maps Tiles")
print("=" * 70)

# ... processing ...

print(f"\n✓ Section 4 complete: {flood_maps.shape}")
```

## Imports Belong in Section 0

All imports go in the first code cell. **Never** scatter imports across the notebook.

### ❌ Bad: imports sprinkled throughout

```python
# Cell 12
import rasterio  # Why wasn't this at the top?
from rasterio.merge import merge as rio_merge
```

### ✅ Good: everything in Section 0

```python
# === Section 0: Environment Setup ===
import numpy as np
import pandas as pd
import xarray as xr
import geopandas as gpd
import rasterio
from rasterio.merge import merge as rio_merge
import matplotlib.pyplot as plt
from pathlib import Path
import logging

# Project imports
from mypackage.geo.hydrobasins import read_vector
from mypackage.models.evt import fit_gpd_model
```

**Exception**: Heavy, rarely-used imports (e.g., `folium`, `cartopy`) can go in the visualization section if they cause import-time slowness.

## End Every Notebook with a Summary Cell

The final cell should report what was produced — files, shapes, key metrics — and point to the next step in the workflow.

```python
print("=" * 70)
print("WORKFLOW COMPLETE")
print("=" * 70)

print(f"\n✓ Outputs created in: {OUTPUT_DIR}")
print(f"  • climada_hazard_{BASIN_ID}.hdf5")
print(f"  • flood-depth_all.nc")
print(f"  • figures/02_flood_depth_maps_by_rp.png")

print(f"\n➡️  Next steps:")
print(f"  1. Open Notebook 03 (Impact Modeling)")
print(f"  2. Load hazard: Hazard.from_hdf5('{hazard_path.name}')")
```
