# 06 — Transform (Business Logic)

## The Transform Function Contract

The transform function is where data scientists write domain logic. It should be a **pure computation** with a clean interface:

```python
def calculate_flood_forecasts(
    data_provider: DataProvider,
    data_submitter: DataSubmitter,
    entity_id: str,
    target_level: int,
) -> None:
    """
    Compute flood forecasts for a given entity.

    Reads input data from data_provider, writes output via data_submitter.
    Does NOT make HTTP calls, read env vars, or access the file system directly.
    """
    # Step 1: Get data
    stations = data_provider.get_data(DataSource.STATIONS, dict)
    admin_areas = data_provider.get_data(DataSource.ADMIN_AREAS, AdminAreasSet)

    # Step 2: Compute
    ...

    # Step 3: Submit results
    data_submitter.create_alert(event_name=..., centroid=...)
    data_submitter.add_severity_data(...)
    data_submitter.add_exposure(...)
```

### Rules for transform functions

| Rule | Rationale |
|------|-----------|
| No `import os`, `import requests`, `from flask import ...` | Domain logic must be framework-free and I/O-free |
| No `os.getenv()` | Configuration comes through the function signature or data provider |
| No file reads/writes | Data comes from `DataProvider`; results go to `DataSubmitter` |
| Type-annotated parameters and return | Enables IDE support and static analysis |
| Docstring explaining what it computes | Data scientists working on this later need context |

## Separate Concerns Within Transform

As transform logic grows, split it into focused modules:

```
flood/
├── __init__.py
├── transform.py              # Main entry point — wires sub-steps
├── return_periods.py          # Compute return period exceedance
├── spatial_aggregation.py     # Aggregate station data to admin areas
└── exposure_calculator.py     # Calculate population exposure
```

Each sub-module is a pure function:

```python
# flood/return_periods.py
def compute_exceedance_probability(
    discharge_data: dict[str, dict[int, dict[str, float]]],
    thresholds: dict[str, float],
) -> dict[str, float]:
    """Compute probability of exceeding threshold for each station."""
    ...
```

## Guard Clauses for Missing Data

Always check that data loaded before proceeding. Fail with a clear message:

```python
def calculate_forecasts(data_provider, data_submitter, entity_id, target_level):
    stations = data_provider.get_data(DataSource.STATIONS, dict)
    admin_areas = data_provider.get_data(DataSource.ADMIN_AREAS, AdminAreasSet)

    # Guard: check data loaded
    if not stations:
        data_submitter.add_error("No station data available")
        return
    if not admin_areas:
        data_submitter.add_error("No admin area data available")
        return

    # Proceed with computation...
```

## Idempotency

Pipeline runs should be **idempotent** — running the same pipeline twice with the same input should produce the same output and not corrupt the target system.

Strategies:
- **Timestamp output** — each run writes to a unique timestamped directory or record
- **Upsert, not insert** — if writing to a database, use upserts keyed on natural identifiers
- **Clear before write** — delete previous results for the same time period before inserting new ones

```python
# Output path includes timestamp for idempotency
timestamp = datetime.now(timezone.utc).strftime("%Y%m%dT%H%M%SZ")
output_path = f"output/{pipeline_type}/{entity_id}/{timestamp}"
```

## Avoid Side Effects

Transform functions should not modify their inputs:

❌ Bad — modifying input DataFrame in-place:
```python
def process(df):
    df["status"] = df["status"].str.lower()  # Mutates caller's data
    df.drop(columns=["temp"], inplace=True)
    return df
```

✅ Good — return new data:
```python
def process(df):
    result = df.copy()
    result["status"] = result["status"].str.lower()
    return result.drop(columns=["temp"])
```

Or better, work with immutable dataclasses:
```python
@dataclass(frozen=True)
class ProcessedStation:
    name: str
    probability: float
    admin_area: str
```

## When to Use pandas

| Situation | Use pandas? |
|-----------|------------|
| Filtering/grouping thousands of rows | ✅ Yes |
| Aggregation (sum, mean, count by group) | ✅ Yes |
| Time series resampling | ✅ Yes |
| Merging/joining two datasets | ✅ Yes |
| Passing 10 records between functions | ❌ Use dataclasses |
| Config objects, API responses | ❌ Use dataclasses |
| Simple iterative computation | ❌ Use plain loops |

## Logging in Transform Functions

Log at meaningful checkpoints, not every line:

```python
def calculate_forecasts(data_provider, data_submitter, entity_id, target_level):
    stations = data_provider.get_data(DataSource.STATIONS, dict)
    logger.info(f"Loaded {len(stations)} stations for {entity_id}")

    # ... computation ...

    logger.info(f"Generated {len(alerts)} alerts for {entity_id}")
    for alert in alerts:
        data_submitter.create_alert(...)
```