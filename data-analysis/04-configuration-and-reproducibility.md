# 04 — Configuration and Reproducibility

## Configuration Belongs in One Cell

The **first code cell after imports** should define all user-configurable parameters. A reader should be able to re-run the notebook for a different region/scenario by editing only this cell.

### ✅ Good: single config cell with clear sections

```python
# === Section 1: Configuration ===

# PART A: User parameters (edit these)
AUTO_DETECT = True
basin_id_input = None          # None = auto-detect, or e.g. "region_01"
run_tag_input = None           # None = auto-detect, or e.g. "2026-01-19_calib-test"

# PART B: Algorithm options
APPLY_FLOPROS = False          # Apply flood protection masking
REGRID_METHOD = "bilinear"     # "bilinear" or "nearest"
USE_RECLASSIFIED = False       # Use reclassified vs raw JRC depth maps

# PART C: Paths (derived from above)
REPO_ROOT = Path.cwd().parent.parent
DATA_ROOT = REPO_ROOT / "data"
PROCESSED_ROOT = DATA_ROOT / "processed"
```

### ❌ Bad: parameters scattered across cells

```python
# Cell 5 (buried in analysis)
threshold = 0.95  # Why is this here?

# Cell 12 (even more buried)
output_format = "parquet"  # Should be at the top
```

## Use UPPER_CASE for Global Config, lower_case for Local Variables

This visual convention makes it immediately clear what's a tunable parameter vs. a computed value.

```python
# Config (upper case) — set once in the config cell
BASIN_ID = "region_01"
RUN_TAG = "2026-01-19_calib-test"
REGRID_METHOD = "bilinear"

# Derived / computed (lower case) — built from config
calibration_root = PROCESSED_ROOT / "calibration" / "evt_pot" / BASIN_ID / RUN_TAG
return_period_nc = calibration_root / "return-period_all.nc"
```

## System Resource Detection

For memory-intensive analysis, detect available RAM and adjust behavior automatically. The reference repo does this well:

```python
def detect_system_resources():
    """Detect available RAM and configure chunk sizes."""
    try:
        import psutil
        mem = psutil.virtual_memory()
        available_gb = mem.available / (1024**3)
        total_gb = mem.total / (1024**3)
    except ImportError:
        available_gb, total_gb = 8.0, 16.0  # Fallback

    low_ram_threshold = 8.0  # GB
    low_ram_mode = available_gb < low_ram_threshold

    if low_ram_mode:
        chunks_xy = (256, 256)
    else:
        chunks_xy = (512, 512)

    return {
        "available_gb": available_gb,
        "total_gb": total_gb,
        "low_ram_mode": low_ram_mode,
        "chunks_xy": chunks_xy,
    }

sys_resources = detect_system_resources()
LOW_RAM_MODE = sys_resources["low_ram_mode"]
DEFAULT_CHUNKS_XY = sys_resources["chunks_xy"]
```

This is a great pattern — someone with 4 GB RAM gets small chunks automatically instead of an OOM crash.

## Configuration Validation

After defining config, validate it. This catches mistakes early instead of 20 minutes into a run.

```python
# Validate all critical config variables
required_vars = {
    "mode": str,
    "BASIN_ID": str,
    "RUN_TAG": str,
    "RETURN_PERIOD_NC": Path,
    "HAZARD_OUTPUT": Path,
}

missing = [var for var in required_vars if var not in globals()]
if missing:
    raise RuntimeError(f"Configuration incomplete: missing {missing}")

# Validate input files exist
if not RETURN_PERIOD_NC.exists():
    raise FileNotFoundError(
        f"Return period file not found: {RETURN_PERIOD_NC}\n"
        f"➡️ Run Notebook 01 first to generate calibration outputs."
    )

print("✅ Configuration validated")
```

## Configuration Usage Map

For complex multi-section notebooks, document which config is used where. The reference repo does this in a dedicated cell:

```python
config_usage = {
    "Section 2": {"uses": ["RETURN_PERIOD_NC", "DEFAULT_CHUNKS_XY"]},
    "Section 3": {"uses": ["mode", "MUNI_GDF", "AOI_BOUNDARY", "USE_RECLASSIFIED"]},
    "Section 4": {"uses": ["DEFAULT_CHUNKS_XY", "NETCDF_ENGINE"]},
    "Section 5": {"uses": ["REGRID_METHOD", "APPLY_FLOPROS", "DEFAULT_CHUNKS_XY"]},
}
```

This is overkill for small notebooks but excellent for complex multi-section workflows where a config change has non-obvious downstream effects.

## Cross-Notebook State via Files, Not Variables

Notebooks in a sequence communicate through files, never through shared kernel state.

### ✅ Good: notebook 02 reads a file

```python
# Notebook 02
if not RETURN_PERIOD_NC.exists():
    raise FileNotFoundError("Run Notebook 01 first")
ds_rp = xr.open_dataset(RETURN_PERIOD_NC, chunks="auto")
```

### ❌ Bad: notebook 02 assumes a variable exists

```python
# Notebook 02
# This assumes notebook 01 was run in the same kernel session
rp_grid = ds_rp["return_period_RP100"]  # NameError if kernel restarted
```

## Save a Run Config JSON

When a notebook produces outputs, save the config that produced them alongside the data. This makes every output traceable.

```python
import json

run_config = {
    "basin_id": BASIN_ID,
    "run_tag": RUN_TAG,
    "selection_mode": mode,
    "algorithm": {
        "regrid_method": REGRID_METHOD,
        "apply_flopros": APPLY_FLOPROS,
    },
    "timestamp": datetime.now().isoformat(),
    "notebook": "02_HazardModeling.ipynb",
}

config_path = HAZARD_OUTPUT / "run_config.json"
config_path.write_text(json.dumps(run_config, indent=2))
print(f"✓ Saved config: {config_path.name}")
```

The reference repo does this — Notebook 01 writes `run_config.json`, and Notebook 02 reads it to auto-detect the calibration mode. This is excellent.