# 01 — Project Structure

## Recommended Layout

The key structural insight from our reference pipelines is the **infra/domain split**: infrastructure code (config parsing, data extraction, output loading) is separated from domain logic (the actual transformation/computation). This pattern scales well and is our recommended default.

There are two proven layouts at 510:

### Layout A: Infra/Domain Split

Best for: pipelines that push results to a single target system, with clear engineer/data-scientist roles.

```
my-pipeline/
├── src/my_pipeline/
│   ├── __init__.py
│   ├── infra/                        # Pipeline infrastructure (maintained by engineers)
│   │   ├── __init__.py
│   │   ├── orchestrator.py           # Main entry point — wires everything together
│   │   ├── config_reader.py          # YAML/JSON config loading and validation
│   │   ├── data_provider.py          # Abstraction over all data sources
│   │   ├── data_submitter.py         # Abstraction over output targets
│   │   ├── data_types/               # Shared typed dataclasses and enums
│   │   │   ├── config_types.py       # Environment, DataSource, OutputMode enums + config dataclasses
│   │   │   ├── domain_types.py       # Domain-specific data structures
│   │   │   └── output_types.py       # Output payload dataclasses
│   │   ├── utils/                    # Helpers: API clients, file loaders, integrity checks
│   │   │   ├── client_<system>.py    # One client module per external system
│   │   │   ├── extract.py            # Per-source extract functions
│   │   │   └── integrity_checks.py   # Output validation before loading
│   │   └── configs/                  # YAML config files per pipeline variant
│   │       ├── floods.yaml
│   │       └── drought.yaml
│   ├── flood/                        # Domain logic: one folder per pipeline variant
│   │   ├── __init__.py
│   │   └── transform.py             # calculate_flood_forecasts(data_provider, data_submitter, ...)
│   └── drought/
│       ├── __init__.py
│       └── transform.py
├── tests/
│   ├── conftest.py
│   ├── unit/                         # Pure logic tests (fast, no I/O)
│   ├── integration_infra/            # Tests pipeline infra with mock/scenario data
│   └── integration_pipeline/         # Full end-to-end tests with controlled inputs
├── Dockerfile
├── pyproject.toml
├── uv.lock
└── README.md
```

### Layout B: Protocol-Based ETL

Best for: pipelines with many heterogeneous data sources, layered storage (bronze/silver/gold), and async I/O.

```
my-pipeline/
├── retrievalpipeline/
│   ├── __init__.py
│   ├── pipeline.py                   # ETLPipeline class + Extractor/Transformer/Loader Protocols
│   ├── config/                       # Country-specific config dataclasses
│   │   ├── __init__.py
│   │   ├── base.py                   # CountryConfig Protocol, DateTimeConfig Protocol
│   │   ├── config.py                 # CONFIGS registry dict
│   │   ├── somalia.py                # SomaliaConfig(CountryConfig)
│   │   └── syria.py
│   ├── extract/
│   │   ├── extractors/               # One class per data source
│   │   │   ├── __init__.py
│   │   │   ├── ecmwf.py              # EcmwfExtractor (async .extract())
│   │   │   ├── worldpop.py
│   │   │   ├── dtm.py
│   │   │   ├── ipc.py
│   │   │   └── extractor_template.py  # Template for new extractors
│   │   ├── constants.py              # StrEnum constants per data source
│   │   └── exceptions/
│   ├── transform/
│   │   └── transformers/             # One class per transformation
│   │       ├── __init__.py
│   │       ├── flood_baseline.py
│   │       ├── drought_recent.py
│   │       └── transformer_template.py
│   ├── load/
│   │   └── load.py                   # Loader Protocol + load() runner
│   └── storage/                      # Storage abstraction (the key pattern)
│       ├── __init__.py
│       ├── base.py                   # Storage ABC: download, store, read, list
│       ├── local_storage.py          # LocalStorage(Storage)
│       └── azure_blob_storage.py     # AzureBlobStorage(Storage)
├── tests/
│   ├── conftest.py                   # Shared fixtures, needs_secrets marker
│   ├── unit/
│   │   ├── test_pipeline.py
│   │   └── extract/
│   │       ├── test_dtm.py
│   │       └── test_worldpop.py
│   └── integration/
│       ├── test_azure_blob_storage.py
│       └── test_ipc_integration.py
├── run_pipeline.py                   # CLI entry point (tyro + RunConfig dataclass)
├── Dockerfile
├── pyproject.toml
├── poetry.lock                       # or uv.lock
└── README.md
```

### Which layout to choose?

| Factor | Layout A (Infra/Domain) | Layout B (Protocol ETL) |
|--------|------------------------|------------------------|
| Pipeline variants | Few (floods, drought) | Many heterogeneous extractors |
| Output target | One system (API, DB) | Layered storage (bronze/silver/gold) |
| Concurrency | Sequential or simple parallel | Heavy async I/O (many APIs) |
| Config style | YAML files | Python dataclasses per country |
| Team roles | Engineer + data scientist | Full-stack data engineers |

## The Infra / Domain Split

This is the most important architectural decision. It enables:

| Benefit | How |
|---------|-----|
| **Data scientists work independently** | They implement one function per pipeline variant, receive a `DataProvider` and `DataSubmitter`, and don't touch config, I/O, or API code |
| **Engineers own reliability** | Config parsing, retries, integrity checks, output formatting — all in `infra/` |
| **Testable at every level** | Infra tests use scenarios (bypass domain logic); domain tests use mock data providers |
| **New variants are cheap** | Copy a folder, implement one function, add a YAML config |

### The contract

Domain logic receives two objects and a context:

```python
def calculate_forecasts(
    data_provider: DataProvider,    # Read-only access to all loaded data sources
    data_submitter: DataSubmitter,  # Write-only interface to build output
    country: str,                   # Context: which route to process
    target_level: int,              # Context: at what granularity
) -> None:
    # 1. Get data from provider
    stations = data_provider.get_data(DataSource.STATIONS, dict)

    # 2. Do domain-specific computation
    alerts = compute_alerts(stations, ...)

    # 3. Submit results via submitter
    for alert in alerts:
        data_submitter.create_alert(...)
```

The domain function:
- **Never** imports config classes, reads env vars, or makes HTTP calls directly
- **Never** knows about the output format (JSON, API, database)
- **Always** receives data through `DataProvider` and pushes results through `DataSubmitter`

## When a Flat Structure Is OK

For one-off scripts or very simple pipelines (single source → single transform → single target):

```
my-script/
├── pipeline.py               # Everything in one file
├── config.yaml               # Pipeline configuration
├── Dockerfile
├── pyproject.toml
└── README.md
```

Graduate to the full structure when:
- You have **more than one pipeline variant** (flood + drought, daily + weekly, etc.)
- Multiple people work on the code (engineer + data scientist)
- The transform logic exceeds ~200 lines

## Anti-Patterns

### Mixing infra and domain

❌ Bad — transform function makes HTTP calls:
```python
def calculate_forecasts(country):
    url = f"https://{os.getenv('API_HOST')}/api/data/{country}"
    response = requests.get(url, headers={"Authorization": os.getenv("API_KEY")})
    data = response.json()
    # ... transform ...
    requests.post(f"https://{os.getenv('API_HOST')}/api/results", json=results)
```

✅ Good — transform function receives and returns data:
```python
def calculate_forecasts(data_provider, data_submitter, country, level):
    data = data_provider.get_data(DataSource.FORECAST_INPUT, ForecastData)
    # ... transform ...
    data_submitter.create_result(event_name=..., value=...)
```

### God-module utils

❌ Bad — one `utils.py` with 500 lines of unrelated functions:
```python
# utils.py
def download_glofas_data(): ...
def parse_admin_areas(): ...
def calculate_return_periods(): ...
def send_to_api(): ...
def format_email(): ...
```

✅ Good — specific, purposeful modules:
```python
# infra/utils/extract.py          — one function per data source
# infra/utils/client_ibf.py       — API interaction, one module per external system
# infra/utils/integrity_checks.py — output validation
# flood/return_periods.py          — domain-specific computation
```