# 03 — Configuration

## YAML-Driven Configuration

Use YAML files to define **what** a pipeline run does, keeping **how** in code. This separation lets operators change targets, data sources, and outputs without modifying Python:

```yaml
# configs/floods.yaml
pipeline_type: floods

run_targets:
  debug:
    entities:
      - id: KEN
        target_level: 3
        data_sources:
          - source: admin_areas_api
          - source: glofas_stations
        output:
          mode: local
          path: output/

  prod:
    entities:
      - id: KEN
        target_level: 3
        data_sources:
          - source: admin_areas_api
          - source: glofas_stations
          - source: glofas_discharge
        output:
          mode: api
      - id: ETH
        target_level: 2
        data_sources:
          - source: admin_areas_api
          - source: glofas_stations
          - source: glofas_discharge
        output:
          mode: api
```

## Run Targets

A run target is a named environment configuration. Use at minimum:

| Run Target | Purpose | Data Sources | Output |
|-----------|---------|-------------|--------|
| `debug` | Local development, quick iteration | Dummy/local data, minimal set | Local files |
| `test` | CI/CD, integration tests | Real-ish data, limited scope | Local files or test API |
| `prod` | Production | All real sources, all entities | Production API/database |

### Why run targets matter

1. **No code changes between environments** — same code, different config
2. **Fast local iteration** — `debug` target loads minimal data
3. **Safe testing** — `test` target talks to staging, not production
4. **Explicit** — what runs where is documented in YAML, not scattered in env vars

## Config Reader with Validation

Parse and validate config at startup. Fail fast with clear error messages:

```python
class ConfigReader:
    def __init__(self):
        self.run_targets: dict[RunTarget, PipelineRunConfig] = {}

    def load(self, path: Path) -> bool:
        """Load and validate config. Returns False on any validation error."""
        try:
            with open(path, "r", encoding="utf-8") as f:
                raw = yaml.safe_load(f)
        except yaml.YAMLError as exc:
            logger.error(f"Invalid YAML: {exc}")
            return False

        # Validate pipeline_type against enum
        try:
            pipeline_type = PipelineType(raw["pipeline_type"])
        except (ValueError, KeyError):
            logger.error(f"Invalid pipeline_type, expected one of {[e.value for e in PipelineType]}")
            return False

        # Validate each run target and entity
        return self._parse_run_targets(raw, pipeline_type)
```

### Validate against enums

Map all string values in the config to enums immediately at parse time:

```python
class DataSource(StrEnum):
    ADMIN_AREAS_API = "admin_areas_api"
    GLOFAS_STATIONS = "glofas_stations"
    GLOFAS_DISCHARGE = "glofas_discharge"
    POPULATION = "population"

class OutputMode(StrEnum):
    LOCAL = "local"
    API = "api"

class RunTarget(StrEnum):
    DEBUG = "debug"
    TEST = "test"
    PROD = "prod"
```

Benefits:
- **Typos caught at startup**: `froods_discharge` fails config parsing, not mid-run
- **Autocomplete**: IDE supports enum values
- **Documentation**: the enum IS the list of valid values

## Environment Variables

Use env vars for **secrets and host-specific values** only. Everything else goes in YAML:

| In YAML | In env vars |
|---------|-------------|
| Which entities to process | `API_KEY` |
| Which data sources to load | `API_HOST` |
| Output mode (local/api) | `DB_PASSWORD` |
| Target granularity level | `APPLICATIONINSIGHTS_CONNECTION_STRING` |

```python
# infra/utils/api_client.py
class ApiClient:
    def __init__(self):
        self.base_url = os.environ.get("API_HOST", "http://localhost:3000")
        api_key = os.environ.get("API_KEY")
        self.headers = {"Authorization": f"Bearer {api_key}"} if api_key else {}
```

### dotenv for local development

```dotenv
# .env — NOT committed
API_HOST=https://staging.example.com
API_KEY=sk-...
```

```dotenv
# example.env — committed, documents all required vars
API_HOST=                   # Base URL of the target API
API_KEY=                    # API key for authentication
```

## Config Dataclasses

Represent parsed config as frozen dataclasses, not dicts:

```python
@dataclass
class DataSourceConfig:
    entity_id: str
    source: DataSource

@dataclass
class EntityRunConfig:
    entity_id: str
    target_level: int
    data_sources: list[DataSourceConfig]
    output_mode: OutputMode
    output_path: str

@dataclass
class PipelineRunConfig:
    run_target: RunTarget
    pipeline_type: PipelineType
    entities: dict[str, EntityRunConfig]
```

❌ Bad — accessing raw dicts throughout the codebase:
```python
country = config["run_targets"]["debug"]["countries"][0]["iso_3_code"]
```

✅ Good — typed, validated config objects:
```python
country = run_config.entities["KEN"].entity_id
```

## CLI Override of Config Values

Allow CLI flags to override config values for flexibility:

```python
@click.command()
@click.option("--config", required=True, type=click.Path(exists=True))
@click.option("--run-target", required=True)
@click.option("--scenario", default=None, help="Override: no-alert or alert")
@click.option("--output-mode", default=None, help="Override output mode")
def main(config, run_target, scenario, output_mode):
    ...
```

The precedence should be: **CLI flags > environment variables > YAML config > defaults**.

## Alternative: Country Config via Protocol + Dataclass

For pipelines where every entity (country) has complex, domain-specific configuration, define a `Protocol` contract and implement it per country using `@dataclass`:

```python
# config/base.py
from typing import ClassVar, Protocol

class CountryConfig(Protocol):
    country_name: ClassVar[str]
    iso3_code: ClassVar[str]
    lat_lon_box: ClassVar[tuple[int, int, int, int]]

    @property
    def drought_data_request(self) -> EcmwfDataRequest: ...

    @property
    def extreme_heat_data_request(self) -> EcmwfDataRequest: ...
```

```python
# config/somalia.py
from dataclasses import dataclass
from typing import ClassVar

@dataclass
class SomaliaConfig:
    continent: ClassVar[str] = "Africa"
    country_name: ClassVar[str] = "Somalia"
    iso3_code: ClassVar[str] = "SOM"
    lat_lon_box: ClassVar[tuple[int, int, int, int]] = (14, 40, -2, 56)

    @property
    def drought_data_request(self) -> EcmwfDataRequest:
        return EcmwfDataRequest(
            collection_id="derived-era5-single-levels-daily-statistics",
            product_type="reanalysis",
            variable="total_precipitation",
            area=list(self.lat_lon_box),
            ...
        )
```

```python
# config/config.py — registry
_CONFIGS: list[type[CountryConfig]] = [SomaliaConfig, SyriaConfig]
CONFIGS: dict[str, CountryConfig] = {c.iso3_code: c() for c in _CONFIGS}
```

### Use `TypedDict` for Component Config

Each extractor/transformer defines its own config type. `TypedDict` is ideal — it's lightweight, typed, and JSON-serializable:

```python
from typing import TypedDict

class DroughtTrendConfig(TypedDict):
    country_iso3: str
    admin_levels: list[int]
    trend_years: list[int]
    lo_q: float
    hi_q: float
    spi_severe: float
```

### When to use which?

| Factor | YAML config (Approach A) | Python config (Approach B) |
|--------|------------------------|---------------------------|
| Operators change config | ✅ YAML is accessible | ❌ Requires Python knowledge |
| Complex derived config | ❌ Limited expressiveness | ✅ Properties, calculations, list comprehensions |
| Type safety | Validated at load time | Validated at definition time (mypy/pyright) |
| Number of entities | Few (2-5 countries) | Many, each with unique parameters |
| Config per component | All in one YAML file | Each component defines its own `TypedDict` |