# 04 — Data Types and Validation

## Use Dataclasses, Not Dicts

The single most impactful rule for pipeline code quality: **never pass raw dicts or JSON through your pipeline**. Convert external data to typed dataclasses as early as possible.

❌ Bad — raw dicts flow through the pipeline:
```python
def process_station(station: dict) -> dict:
    name = station["properties"]["stationName"]     # KeyError at runtime
    lat = station["geometry"]["coordinates"][1]      # IndexError? Wrong axis?
    return {"name": name, "lat": lat, "level": station["data"]["waterLevel"]}
```

✅ Good — parse to dataclass at the boundary, use typed objects everywhere:
```python
@dataclass
class Station:
    name: str
    latitude: float
    longitude: float
    water_level: float

    @classmethod
    def from_api(cls, raw: dict) -> "Station":
        """Parse from API response. All validation happens here."""
        props = raw["properties"]
        coords = raw["geometry"]["coordinates"]
        return cls(
            name=props["stationName"],
            latitude=coords[1],
            longitude=coords[0],
            water_level=raw["data"]["waterLevel"],
        )
```

### Benefits

| Benefit | Explanation |
|---------|-------------|
| **Autocomplete** | IDE shows available fields — no guessing key names |
| **Type checking** | `mypy` / `pyright` catches type mismatches before runtime |
| **Parse errors at the boundary** | Malformed data fails at import time, not deep in transform logic |
| **Easy to create with LLMs** | Give the LLM a sample API response; it generates the dataclass. |
| **Refactor-safe** | Rename a field → IDE/linter finds all usages |

## Enum All The Things

Every categorical value should be an enum:

```python
from enum import StrEnum

class DataSource(StrEnum):
    ADMIN_AREAS = "admin_areas"
    STATIONS = "stations"
    FORECAST = "forecast"
    POPULATION = "population"

class OutputMode(StrEnum):
    LOCAL = "local"
    API = "api"

class HazardType(StrEnum):
    FLOODS = "floods"
    DROUGHT = "drought"

class Layer(StrEnum):
    ALERT_EXTENT = "alert_extent"
    POPULATION_EXPOSED = "population_exposed"
```

Use `StrEnum` so enums serialize to strings naturally (no `.value` needed in most contexts).

## Loaded Data Container

Wrap loaded data in a container that tracks its type, source, and any errors:

```python
@dataclass
class LoadedDataSource:
    data_type: DataType
    data_source: DataSource
    data: object | None = None
    error: str | None = None
    metadata: dict[str, str | int | float | bool] = field(default_factory=dict)
```

Then the `DataProvider` can do runtime type checking when domain code requests data:

```python
def get_data(self, source: DataSource, expected_type: type[T]) -> T:
    container = self.loaded_data[source]
    if not isinstance(container.data, expected_type):
        raise TypeError(
            f"'{source}' expected {expected_type.__name__}, got {type(container.data).__name__}"
        )
    return container.data
```

This catches data-type mismatches immediately, with clear messages.

## Output Type Hierarchy

Build output payloads as typed dataclasses with `to_dict()` methods for serialization:

```python
@dataclass
class Centroid:
    latitude: float
    longitude: float

    def to_dict(self) -> dict[str, float]:
        return {"latitude": self.latitude, "longitude": self.longitude}

@dataclass
class Result:
    event_name: str
    centroid: Centroid
    severity: list[Severity] = field(default_factory=list)
    exposure: Exposure = field(default_factory=Exposure)

    def to_dict(self) -> dict:
        return {
            "eventName": self.event_name,
            "centroid": self.centroid.to_dict(),
            "severity": [s.to_dict() for s in self.severity],
            "exposure": self.exposure.to_dict(),
        }
```

### Why `to_dict()` instead of `dataclasses.asdict()`?

- **camelCase output** — APIs often expect camelCase; Python uses snake_case
- **Custom serialization** — dates, enums, nested objects need explicit handling
- **Performance** — `asdict()` deep-copies recursively; `to_dict()` is explicit

## Integrity Checks

Validate output **before** sending it to the target system. Write focused check functions:

```python
def check_event_name_format(event_name: str) -> list[str]:
    """Validate event name follows pattern: {ISO3}_{type}_{identifier}."""
    if not EVENT_NAME_PATTERN.match(event_name):
        return [f"Invalid event name format: '{event_name}'"]
    return []

def check_centroid(event_name: str, centroid: Centroid) -> list[str]:
    errors = []
    if not -90 <= centroid.latitude <= 90:
        errors.append(f"'{event_name}': latitude {centroid.latitude} out of range")
    if not -180 <= centroid.longitude <= 180:
        errors.append(f"'{event_name}': longitude {centroid.longitude} out of range")
    return errors

def check_severity_integrity(event_name: str, result: Result) -> list[str]:
    errors = []
    if not result.severity:
        errors.append(f"'{event_name}' has no severity data")
    for entry in result.severity:
        if entry.time_interval.start >= entry.time_interval.end:
            errors.append(f"'{event_name}': time interval start >= end")
    return errors
```

Compose them in the submitter:
```python
def _check_integrity(self) -> list[str]:
    errors = []
    for name, result in self._results.items():
        errors.extend(check_event_name_format(name))
        errors.extend(check_centroid(name, result.centroid))
        errors.extend(check_severity_integrity(name, result))
    return errors
```

### Integrity checks should mirror target validation

If the target API validates payloads, your integrity checks should replicate that validation. This way, errors are caught locally with clear messages, instead of getting a cryptic `422 Unprocessable Entity` from the API.

## Avoid pandas for Everything

Pandas is excellent for tabular data manipulation but is overkill — and often harmful — for:

| Use case | Better alternative |
|----------|-------------------|
| Passing structured records between functions | Dataclasses / lists of dataclasses |
| Config objects | Dataclasses |
| Small lookups (< 1000 items) | `dict` |
| API response parsing | Dataclasses with `from_api()` |
| Type-safe domain objects | Dataclasses |

Use pandas when you genuinely need:
- Vectorized numeric operations on large arrays
- GroupBy / merge / pivot operations
- Time series resampling
