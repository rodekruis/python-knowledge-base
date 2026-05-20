# 02 — Pipeline Architecture

## The ETL Flow

Every pipeline follows the same high-level flow:

```
┌──────────┐     ┌─────────────┐     ┌──────────┐
│ EXTRACT  │────▶│  TRANSFORM  │────▶│   LOAD   │
│          │     │             │     │          │
│ Fetch    │     │ Domain      │     │ Validate │
│ data     │     │ logic       │     │ output   │
│ sources  │     │ (compute,   │     │ Submit   │
│          │     │  aggregate) │     │ to target│
└──────────┘     └─────────────┘     └──────────┘
     ▲                                     │
     │         ┌─────────────┐             │
     └─────────│   CONFIG    │─────────────┘
               │             │
               │ What to     │
               │ fetch,      │
               │ where to    │
               │ send        │
               └─────────────┘
```

## Orchestrator Pattern

There are two proven orchestration approaches at 510. Choose based on your pipeline's complexity.

### Approach A: Function-Based Orchestrator

The orchestrator reads config, creates provider and submitter, and runs the pipeline per entity:

```python
# infra/orchestrator.py

def run_pipeline(config_path: str, run_target: str) -> list[str]:
    """Run the pipeline. Returns a list of error messages (empty = success)."""
    config = ConfigReader()
    if not config.load(config_path):
        return ["Failed to load config"]

    run_config = config.get_run_target(run_target)
    transform_fn = TRANSFORM_FUNCTIONS[run_config.pipeline_type]

    all_errors: list[str] = []
    for entity in run_config.entities:
        provider = DataProvider(api_client)
        provider.load_data(config, entity, run_target)
        submitter = DataSubmitter(api_client)
        transform_fn(provider, submitter, entity.id, entity.target_level)
        all_errors.extend(submitter.send_all(entity.output_mode, entity.output_path))
    return all_errors
```

### Approach B: Protocol-Based ETLPipeline Class

Define component contracts with `Protocol`, then compose them into a typed pipeline:

```python
from typing import Protocol

class Extractor(Protocol):
    name: str
    async def extract(self) -> None: ...

class Transformer(Protocol):
    name: str
    async def transform(self) -> None: ...

class Loader(Protocol):
    name: str
    async def load(self) -> None: ...

class ETLPipeline:
    def __init__(
        self,
        extractors: list[Extractor] | None = None,
        transforms: list[Transformer] | None = None,
        loaders: list[Loader] | None = None,
    ):
        self.extractors = extractors or []
        self.transforms = transforms or []
        self.loaders = loaders or []

    def extract(self) -> None:
        extracts = [e.extract() for e in self.extractors]
        asyncio.run(self._run_steps(extracts, [e.name for e in self.extractors], "extractions"))

    def transform(self) -> None:
        transforms = [t.transform() for t in self.transforms]
        asyncio.run(self._run_steps(transforms, [t.name for t in self.transforms], "transforms"))

    @staticmethod
    async def _run_steps(steps, names, label) -> None:
        results = await asyncio.gather(*steps, return_exceptions=True)
        errors = [(n, r) for n, r in zip(names, results) if isinstance(r, Exception)]
        if errors:
            for name, err in errors:
                logger.error(f"{name} failed: {err}")
            raise ExceptionGroup(f"{label}-failed", [e for _, e in errors])
```

Then the entry point **constructs** the pipeline from typed components:

```python
def construct_pipeline(run_id: str, iso3: str) -> ETLPipeline:
    config = CONFIGS[iso3]
    storage = AzureBlobStorage("my-container")  # or LocalStorage() for dev

    extractors = [
        EcmwfExtractor(name="Drought", storage=storage, config=config.drought_data_request, ...),
        DtmExtractor(name="DTM", storage=storage, config=config.dtm_data_request, ...),
    ]
    transforms = [
        DroughtBaselineTransformer(name="Drought Baseline", storage=storage, config=..., ...),
        FloodRecentTransformer(name="Flood Recent", storage=storage, config=..., ...),
    ]
    return ETLPipeline(extractors, transforms, loaders=[])
```

**Why this is powerful**: every component receives `storage` at construction — you swap `LocalStorage()` for `AzureBlobStorage()` in one place and the entire pipeline changes behavior.

## DataProvider — The Extract Layer

The `DataProvider` abstracts all data sources behind a uniform interface:

```python
class DataProvider:
    def __init__(self, api_client: ApiClient):
        self.loaded_data: dict[DataSource, LoadedDataSource] = {}

    def load_data(self, config, entity, run_target) -> bool:
        """Load all data sources configured for this entity. Returns False if any critical source fails."""
        for source_config in entity.data_sources:
            container = LoadedDataSource(data_source=source_config.source)
            try:
                load_data_container(entity, source_config, container, self.api_client)
            except Exception as exc:
                container.error = str(exc)
                logger.error(f"Failed to load {source_config.source}: {exc}")
            self.loaded_data[source_config.source] = container
        return True

    def get_data(self, source: DataSource, expected_type: type[T]) -> T:
        """Get loaded data with runtime type checking."""
        if source not in self.loaded_data:
            raise KeyError(f"Data source '{source}' not loaded")
        container = self.loaded_data[source]
        if not isinstance(container.data, expected_type):
            raise TypeError(f"Expected {expected_type.__name__}, got {type(container.data).__name__}")
        return container.data
```

### Adding a new data source

1. Add an enum value to `DataSource`
2. Write a fetch function in `data_fetchers.py`
3. Register it in the `load_data_container` dispatch (match/case or dict lookup)
4. Add the source to the YAML config

This pattern keeps fetch logic isolated per source and easy to test individually.

## DataSubmitter — The Load Layer

The `DataSubmitter` is a builder that accumulates output, validates it, then sends it:

```python
class DataSubmitter:
    def __init__(self, api_client: ApiClient):
        self._results: list[Result] = {}
        self.errors: dict[str, str] = {}

    def create_result(self, ...):
        """Domain code calls this to build output incrementally."""
        ...

    def send_all(self, output_mode: OutputMode, output_path: str) -> list[str]:
        """Validate and send all accumulated results. Returns error list."""
        # 1. Run integrity checks BEFORE sending
        errors = self._check_integrity()
        if errors:
            return errors

        # 2. Send to target
        match output_mode:
            case OutputMode.API:
                return self._send_to_api()
            case OutputMode.LOCAL:
                return self._write_to_file(output_path)
```

### Why validate before sending?

Catching malformed output before it hits the target system:
- Saves API calls and bandwidth
- Produces clearer error messages (your integrity check vs a cryptic 400 response)
- Prevents partial writes (either all results are valid, or none are sent)

## Scenario-Based Testing

Approach A introduces **scenarios** — synthetic overrides that replace domain logic with predetermined output. This is an excellent testability pattern:

```
┌────────────────────────────────┐
│  run_target   ×   scenario     │
├────────────────────────────────┤
│  DEBUG        ×   (none)       │  → Real domain logic, debug data sources
│  DEBUG        ×   no-alert     │  → Infra-only test: empty output
│  DEBUG        ×   alert        │  → Infra-only test: synthetic output
│  PROD         ×   (none)       │  → Production run
└────────────────────────────────┘
```

## Storage Abstraction

**This is one of the most important patterns for data pipelines.** Define a `Storage` ABC that extractors and transformers depend on, then swap implementations at construction time:

```python
from abc import ABC, abstractmethod

class Storage(ABC):
    @abstractmethod
    async def download(self, url: str, destination: str) -> None: ...

    @abstractmethod
    async def store_chunks(self, chunks, destination: str) -> None: ...

    @abstractmethod
    async def read_data(self, source: str) -> bytes: ...

    @abstractmethod
    async def list_files(self, prefix: str = "", suffix: str = "") -> list[str]: ...

    # Concrete methods built on top of abstracts
    async def store_csv(self, df, destination: str) -> None:
        csv_bytes = df.to_csv(index=False, sep=";").encode()
        await self.store(csv_bytes, destination)

    async def download_multiple(self, urls_destinations) -> None:
        semaphore = asyncio.Semaphore(self.download_concurrency)
        async with ClientSession() as session, asyncio.TaskGroup() as tg:
            for url, path in urls_destinations:
                tg.create_task(self._guarded_download(url, path, session, semaphore))
```

Then provide two implementations:

```python
class LocalStorage(Storage):
    """For development and testing."""
    async def store_chunks(self, chunks, destination, overwrite=True):
        target = Path(destination)
        target.parent.mkdir(parents=True, exist_ok=True)
        async with aiofiles.open(target, "wb") as f:
            async for chunk in chunks:
                await f.write(chunk)

class AzureBlobStorage(Storage):
    """For production."""
    def __init__(self, container_name: str):
        self.container = container_name
        self.blob_config = _get_blob_service_client_config()

    async def store_chunks(self, chunks, destination, overwrite=True):
        async with BlobServiceClient(**self.blob_config) as client:
            container = client.get_container_client(self.container)
            await container.upload_blob(name=destination, data=chunks, overwrite=overwrite)
```

### Why this matters

| Benefit | Without Storage ABC | With Storage ABC |
|---------|-------------------|-----------------|
| Local development | Must mock Azure SDK | Just use `LocalStorage()` |
| Unit testing | Complex mock setup | `MagicMock(spec=Storage)` |
| Adding S3 support | Rewrite all extractors | Add one `S3Storage(Storage)` class |
| Reading extracted data | Each transformer knows about Azure/local | `await storage.read_data(path)` works everywhere |

## Bronze / Silver / Gold Layers

Name your storage paths by data maturity:

| Layer | Purpose | Example path |
|-------|---------|-------------|
| **Bronze** | Raw extracted data, exactly as received | `data/bronze/somalia/ecmwf/utci/` |
| **Silver** | Cleaned, transformed, admin-level aggregated | `data/silver/somalia/climate/drought/` |
| **Gold** | Final output ready for consumption | `data/gold/somalia/indicators/` |

```python
path_to_bronze = f"data/bronze/{config.country_name.lower()}"
path_to_silver = f"data/silver/{config.country_name.lower()}"
path_to_gold   = f"data/gold/{config.country_name.lower()}"
```