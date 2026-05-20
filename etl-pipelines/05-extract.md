# 05 — Extract (Data Fetching)

## One Function Per Data Source

Each data source gets its own fetch function. This keeps fetch logic isolated, testable, and easy to understand:

```python
# infra/utils/data_fetchers.py

def load_data_container(entity_config, source_config, container, api_client):
    """Dispatch to the appropriate loader based on data source type."""
    match source_config.source:
        case DataSource.ADMIN_AREAS:
            _load_admin_areas(container, api_client, source_config.entity_id)
        case DataSource.STATIONS:
            _load_stations(source_config, container)
        case DataSource.FORECAST:
            _load_forecast(source_config, container)
        case _:
            raise ValueError(f"Unknown source: {source_config.source}")


def _load_admin_areas(container, api_client, entity_id):
    """Fetch admin areas from the API."""
    container.data_type = DataType.ADMIN_AREA_SET
    data = api_client.get_admin_areas(entity_id)
    container.data = AdminAreasSet.from_api(data)


def _load_stations(source_config, container):
    """Load station data from the seed data repository."""
    container.data_type = DataType.LOCATION_POINTS
    raw = download_json_source(f"{SEED_REPO_URI}/stations/{source_config.entity_id}.json")
    container.data = {s["id"]: LocationPoint.from_dict(s) for s in raw}
```

## Data Source Types

Data comes from different kinds of sources. Handle each appropriately:

| Source type | Pattern | Example |
|------------|---------|---------|
| REST API | `requests.get()` with auth headers | 121 API, GloFAS API |
| File download (URL) | `requests.get()` + parse | CSV/JSON from a static URL, seed data repo |
| Blob storage | Azure SDK / `requests` with SAS token | Azure Blob, S3 |
| Local file | `Path.open()` | Mapping CSVs, GeoJSON files |
| Database | Connection pool + query | PostGIS, Azure SQL |

## HTTP Requests: Retry and Timeout

Always set timeouts and implement retries for external HTTP calls:

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

def create_resilient_session() -> requests.Session:
    """Create a requests session with retry logic and timeouts."""
    session = requests.Session()

    retries = Retry(
        total=3,
        backoff_factor=1,            # 1s, 2s, 4s
        status_forcelist=[429, 500, 502, 503, 504],
        allowed_methods=["GET"],     # Only retry idempotent methods
    )
    adapter = HTTPAdapter(max_retries=retries)
    session.mount("https://", adapter)
    session.mount("http://", adapter)

    return session


# Usage
session = create_resilient_session()
response = session.get(url, timeout=30, headers=headers)
response.raise_for_status()
```

❌ Bad — no timeout, no retries:
```python
response = requests.get(url)  # Hangs forever if server is slow
data = response.json()        # Crashes on non-200 without explanation
```

✅ Good — timeout + retry + explicit error:
```python
response = session.get(url, timeout=30)
response.raise_for_status()
data = response.json()
```

## Parse at the Boundary

Convert raw API/file data to typed dataclasses **immediately** after fetching:

```python
def _load_admin_areas(container, api_client, entity_id):
    raw = api_client.get_admin_areas(entity_id)     # Returns raw dict/JSON
    container.data = AdminAreasSet.from_api(raw)     # Parsed to dataclass HERE
    container.data_type = DataType.ADMIN_AREA_SET
```

This means:
- Parse errors surface at extract time, with the source context clear
- Domain code only ever sees typed objects
- You can unit-test parsing separately from fetching

## Paginated APIs

For APIs that paginate results, fetch all pages transparently:

```python
def fetch_all_pages(session, url, headers, *, page_size=1000) -> list[dict]:
    """Fetch all pages from a paginated API."""
    all_data = []
    offset = 0

    while True:
        params = {"limit": page_size, "offset": offset}
        response = session.get(url, headers=headers, params=params, timeout=30)
        response.raise_for_status()

        batch = response.json()["data"]
        all_data.extend(batch)

        if len(batch) < page_size:
            break  # Last page
        offset += page_size

    logger.info(f"Fetched {len(all_data)} records from {url}")
    return all_data
```

## Dummy / Stub Data for Early Development

During early prototyping, use dummy data that matches the real shape:

```python
DUMMY_DATA: dict[DataSource, object] = {
    DataSource.FORECAST: {
        "station-A": {
            0: {"member-1": 80, "member-2": 85},
            1: {"member-1": 90, "member-2": 95},
        },
    },
}

def _load_dummy_data(source_config):
    return DUMMY_DATA.get(source_config.source)
```

Approach A uses this pattern explicitly. The key rules for dummy data:

1. **Same shape** as real data (same keys, same nesting)
2. **Clearly marked** — use a `TODO_` prefix on enum values or a dedicated module
3. **Replaced before production** — tracked as tech debt

## Caching / Avoiding Re-downloads

For large datasets that don't change frequently, consider local caching:

```python
def _load_with_cache(url: str, cache_dir: Path, ttl_hours: int = 24) -> bytes:
    """Download and cache a file locally."""
    cache_file = cache_dir / hashlib.sha256(url.encode()).hexdigest()

    if cache_file.exists():
        age_hours = (time.time() - cache_file.stat().st_mtime) / 3600
        if age_hours < ttl_hours:
            logger.info(f"Using cached file for {url}")
            return cache_file.read_bytes()

    logger.info(f"Downloading {url}")
    response = session.get(url, timeout=60)
    response.raise_for_status()

    cache_dir.mkdir(parents=True, exist_ok=True)
    cache_file.write_bytes(response.content)
    return response.content
```

Use this for:
- Large static reference data (admin boundaries, population rasters)
- Data that changes at most daily

**Never** cache data that changes between runs (forecast data, real-time sensor readings).

## Async Extractors with Storage Abstraction

For pipelines with many data sources, make extractors async and inject a `Storage` instance. The extractor class doesn't know whether it's writing to local disk or Azure Blob:

```python
class EcmwfExtractor:
    def __init__(self, name: str, storage: Storage, path_to_output: str, data_request: EcmwfDataRequest):
        self.name = name
        self.storage = storage
        self.path_to_output = Path(path_to_output)
        self.data_request = data_request

    async def extract(self) -> None:
        """Submit CDS request, wait for results, download to storage."""
        remote = self.client.submit(self.collection_id, self.request)
        await self._wait_on_results(remote)
        url = remote.get_results().location
        await self.storage.download(url, str(self.path_to_output))
```

### Concurrent downloads with `TaskGroup`

```python
async def extract(self) -> None:
    df_selected = self.select_files(self.get_available_files())
    metadata = self.select_metadata(df_selected)

    async with asyncio.TaskGroup() as tg:
        tg.create_task(self.storage.store_csv(df_selected, str(self.path_to_data)))
        tg.create_task(self.storage.store_csv(metadata, str(self.path_to_metadata)))
```

## Validate API Responses with Pydantic

External APIs return unpredictable data. Validate responses immediately using Pydantic models:

```python
from pydantic import BaseModel, Field

class DtmAdminAreaSchema(BaseModel):
    id: int
    admin1_pcode: str
    admin2_pcode: str = ""
    num_presences: int = Field(alias="numPresences", default=0)
    round_number: int = Field(alias="roundNumber")

class ApiJsonExtractor:
    """Validate API responses against a Pydantic schema."""

    def __init__(self, schema: type[BaseModel], data_field: str | None = None):
        self.schema = schema
        self.data_field = data_field

    def get_data(self, api_url: str, params=None, headers=None) -> list[BaseModel]:
        response = self.get_response(api_url, params, headers)
        raw_data = response.json()
        if self.data_field:
            raw_data = raw_data[self.data_field]

        validated = []
        for item in raw_data:
            try:
                validated.append(self.schema.model_validate(item))
            except ValidationError:
                logger.warning(f"Skipping invalid record: {item}")
        return validated
```

### Why validate at the boundary?

- **Bad data from APIs is the #1 cause of pipeline failures.** Field names change, types shift, nullable fields appear.
- **Pydantic catches it immediately** with clear error messages, instead of a cryptic `KeyError` deep in your transform logic.
- **Invalid records are logged and skipped**, so one bad record doesn't crash the entire pipeline.

## Component Templates

Provide templates for new extractors/transformers so developers get the structure right from the start:

```python
# extract/extractors/extractor_template.py

class SomeDataExtractor:
    """Extractor for files from some data server."""

    def __init__(self, name: str, storage: Storage, path_to_output: str, data_request: SomeDataRequest):
        self.name = name
        self.storage = storage
        self.path_to_output = Path(path_to_output)
        self.data_specification = data_request["data_specification"]

    async def extract(self) -> None:
        file_urls = self._get_file_urls()
        urls_paths = self.get_urls_and_paths(self.path_to_output, file_urls)
        metadata = self.get_metadata()

        async with asyncio.TaskGroup() as tg:
            tg.create_task(self.storage.store_csv(metadata, str(self.path_to_metadata)))
            tg.create_task(self.storage.download_multiple(urls_paths))
```