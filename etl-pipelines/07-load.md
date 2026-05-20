# 07 — Load (Output)

## Output Modes

Support multiple output modes controlled by config, not hardcoded:

```python
class OutputMode(StrEnum):
    LOCAL = "local"      # Write JSON/CSV to local file system
    API = "api"          # POST to target API
    # Future: DATABASE, BLOB_STORAGE, etc.
```

The `DataSubmitter` dispatches based on mode:

```python
def send_all(self, output_mode: OutputMode, output_path: str) -> list[str]:
    # 1. Validate BEFORE sending
    errors = self._check_integrity()
    if errors:
        logger.error(f"Integrity check failed: {errors}")
        return errors

    # 2. Dispatch to target
    match output_mode:
        case OutputMode.LOCAL:
            return self._write_to_file(output_path)
        case OutputMode.API:
            return self._send_to_api()
```

## Validate Before Sending

**Always** run integrity checks before submitting to the target:

```python
def _check_integrity(self) -> list[str]:
    """Validate all accumulated results. Returns list of errors."""
    errors = []

    # Check metadata
    if self._metadata is None:
        errors.append("Missing pipeline metadata (issued_at, type)")
        return errors  # Can't validate further without metadata

    # Check each result
    for name, result in self._results.items():
        errors.extend(check_event_name_format(name))
        errors.extend(check_centroid(name, result.centroid))
        errors.extend(check_severity_integrity(name, result))
        errors.extend(check_exposure_integrity(name, result))

    return errors
```

### Why this matters

| Without pre-validation | With pre-validation |
|----------------------|-------------------|
| API returns `422 Unprocessable Entity` | Clear message: "latitude 91.3 out of range" |
| Partial writes (some results sent, some rejected) | All-or-nothing: either everything is valid, or nothing is sent |
| Error message from remote system, often cryptic | Error message from your code, with full context |
| One round-trip per validation error | All errors caught locally in one pass |

## Local File Output

For development and debugging, write results as JSON files with timestamped directories:

```python
def _write_to_file(self, output_dir: str) -> list[str]:
    os.makedirs(output_dir, exist_ok=True)

    # Write full payload
    payload = self._build_payload()
    output_file = os.path.join(output_dir, "forecast.json")
    with open(output_file, "w", encoding="utf-8") as f:
        json.dump(payload, f, indent=2, ensure_ascii=False)

    logger.info(f"Wrote output to {output_file}")
    return []
```

Directory structure:
```
output/
├── floods/
│   └── KEN/
│       ├── 20260513T120000Z/
│       │   └── forecast.json
│       └── 20260514T120000Z/
│           └── forecast.json
└── drought/
    └── ETH/
        └── 20260513T120000Z/
            └── forecast.json
```

## API Output

For production, POST results to the target API:

```python
def _send_to_api(self) -> list[str]:
    payload = self._build_payload()

    try:
        response = self.api_client.post_results(payload)
        if response.status_code not in range(200, 300):
            return [f"API returned {response.status_code}: {response.text}"]
        logger.info("Successfully submitted results to API")
        return []
    except requests.RequestException as exc:
        return [f"API submission failed: {exc}"]
```

### API submission best practices

1. **Single payload** — batch all results into one request when the API supports it, rather than one request per result
2. **Idempotent endpoints** — prefer PUT/PATCH with natural keys over POST that creates duplicates
3. **Check response body** — some APIs return 200 with error details in the body
4. **Log the request ID** — if the API returns a request/correlation ID, log it for traceability

## Atomic Writes

Whether writing to files or an API, aim for **atomic writes** — either everything succeeds or nothing is committed:

### File output
```python
import tempfile
import shutil

def _write_atomic(self, output_dir: str, payload: dict) -> None:
    """Write to a temp file first, then move to final location."""
    os.makedirs(output_dir, exist_ok=True)

    with tempfile.NamedTemporaryFile(mode="w", suffix=".json", delete=False, dir=output_dir) as tmp:
        json.dump(payload, tmp, indent=2, ensure_ascii=False)
        tmp_path = tmp.name

    final_path = os.path.join(output_dir, "forecast.json")
    shutil.move(tmp_path, final_path)
```

### API output
- Use transaction endpoints if available
- If the API doesn't support transactions, validate thoroughly before sending to minimize partial failure risk

## Output Schema Documentation

Document the exact shape of your output. If the target API has an OpenAPI spec, keep your output types in sync:

```python
@dataclass
class Forecast:
    """Output payload. Schema must match the target API's ForecastCreateDto."""
    issued_at: datetime
    pipeline_type: str
    sources: list[str]
    results: list[Result]

    def to_dict(self) -> dict:
        return {
            "issuedAt": self.issued_at.strftime("%Y-%m-%dT%H:%M:%SZ"),
            "hazardType": self.pipeline_type,
            "forecastSources": self.sources,
            "alerts": [r.to_dict() for r in self.results],
        }
```

> **Keep pipeline output types in sync with target API DTOs.** If the API team changes a field name, your integrity checks should catch the mismatch in CI before it hits production.