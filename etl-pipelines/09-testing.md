# 09 — Testing

## Test Pyramid for Pipelines

```
tests/
├── conftest.py                    # Shared fixtures
├── unit/                          # Pure logic, no I/O (fast)
│   ├── test_integrity_checks.py
│   ├── test_config_reader.py
│   ├── test_return_periods.py     # Domain-specific computation
│   └── test_data_submitter.py
├── integration_infra/             # Pipeline infra with scenarios (medium)
│   ├── conftest.py                # Pipeline runner fixture
│   ├── test_floods.py
│   └── test_drought.py
└── integration_pipeline/          # Full pipeline with mock data (slow)
    └── test_flood_end_to_end.py
```

| Tier | What it tests | Speed | External deps |
|------|--------------|-------|---------------|
| Unit | Individual functions: integrity checks, config parsing, domain math | ⚡ ms | None |
| Integration (infra) | Full infra path using scenarios (bypasses domain logic) | 🐢 seconds | May need test API |
| Integration (pipeline) | Full pipeline with controlled mock input data | 🐌 slower | May need test API + data files |

## Unit Tests

### Integrity checks

```python
def test_valid_event_name():
    errors = check_event_name_format("KEN_floods_station-A")
    assert errors == []

def test_invalid_event_name():
    errors = check_event_name_format("invalid-name")
    assert len(errors) == 1
    assert "Invalid event name" in errors[0]

def test_centroid_out_of_range():
    centroid = Centroid(latitude=91.0, longitude=0.0)
    errors = check_centroid("TEST_floods_x", centroid)
    assert any("latitude" in e for e in errors)
```

### Config reader

```python
def test_config_loads_valid_yaml(tmp_path):
    config_file = tmp_path / "test.yaml"
    config_file.write_text("""
pipeline_type: floods
run_targets:
  debug:
    countries:
      - iso_3_code: KEN
        target_admin_level: 3
        data_sources:
          - source: admin_areas
        output:
          mode: local
          path: output/
""")
    reader = ConfigReader()
    assert reader.load(config_file) is True
    assert RunTarget.DEBUG in reader.run_targets

def test_config_rejects_invalid_pipeline_type(tmp_path):
    config_file = tmp_path / "bad.yaml"
    config_file.write_text("pipeline_type: invalid\nrun_targets: {}")
    reader = ConfigReader()
    assert reader.load(config_file) is False
```

### Domain logic

Test domain functions with mock data providers:

```python
def test_return_period_calculation():
    discharge = {
        "station-A": {0: {"m1": 100, "m2": 200, "m3": 300}},
    }
    thresholds = {"station-A": 150}

    result = compute_exceedance_probability(discharge, thresholds)
    assert 0.0 <= result["station-A"] <= 1.0
    assert result["station-A"] == pytest.approx(2 / 3)  # 2 of 3 members exceed 150
```

## Scenario-Based Integration Tests

Approach A’s best testing pattern: **scenarios** that bypass domain logic to test infrastructure in isolation.

```python
# tests/integration_infra/conftest.py

@dataclass(frozen=True)
class PipelineHelpers:
    run_pipeline: Callable[..., subprocess.CompletedProcess[str]]
    load_output: Callable[..., dict]
    assert_output_structure: Callable[..., None]
    clean_output: Callable[..., None]

@pytest.fixture()
def pipeline() -> PipelineHelpers:
    return PipelineHelpers(
        run_pipeline=_run_pipeline,
        load_output=_load_output,
        assert_output_structure=_assert_output_structure,
        clean_output=_clean_output,
    )

def _run_pipeline(config, run_target, *, scenario=None, extra_env=None):
    """Run the pipeline as a subprocess."""
    cmd = [sys.executable, "-m", "my_pipeline.infra.orchestrator",
           "--config", config, "--run-target", run_target]
    if scenario:
        cmd.extend(["--scenario", scenario])

    env = os.environ.copy()
    if extra_env:
        env.update(extra_env)

    return subprocess.run(cmd, env=env, capture_output=True, text=True)
```

### Infrastructure tests

```python
def test_pipeline_no_alert_scenario(pipeline):
    """Run pipeline with no-alert scenario. Tests: config → extract → validation → output."""
    result = pipeline.run_pipeline(
        "configs/floods.yaml", "DEBUG",
        scenario="no-alert",
    )
    assert result.returncode == 0, f"Failed:\n{result.stderr}"

def test_pipeline_alert_scenario(pipeline):
    """Run pipeline with alert scenario. Tests: full infra path with synthetic alerts."""
    result = pipeline.run_pipeline(
        "configs/floods.yaml", "DEBUG",
        scenario="alert",
        extra_env={"OUTPUT_MODE": "local"},
    )
    assert result.returncode == 0

    output = pipeline.load_output("floods", "KEN")
    pipeline.assert_output_structure(output)
```

### What scenarios test

| Scenario | Domain logic runs? | Tests |
|----------|-------------------|-------|
| `no-alert` | ❌ Bypassed | Config parsing, data loading, empty output submission |
| `alert` | ❌ Bypassed | Config parsing, data loading, synthetic output, integrity checks, submission |
| _(none)_ | ✅ Full run | Everything including domain computation |

## Mocking External APIs

For unit tests that touch API client code, mock the HTTP layer:

```python
import responses

@responses.activate
def test_api_client_get_admin_areas():
    responses.add(
        responses.GET,
        "https://api.example.com/admin-areas?country=KEN",
        json=[{"pcode": "KE01", "name": "Nairobi", "admin_level": 1}],
        status=200,
    )

    client = ApiClient(base_url="https://api.example.com")
    result = client.get_admin_areas("KEN")
    assert len(result) == 1
    assert result[0]["pcode"] == "KE01"
```

## Fixtures for Test Data

Create reusable test data factories:

```python
# tests/conftest.py

@pytest.fixture
def sample_admin_areas():
    return AdminAreasSet(admin_areas={
        "KE01": AdminArea(
            properties=AdminAreaProperties(name="Nairobi", pcode="KE01", admin_level=1, ...),
            coordinates=[],
        ),
    })

@pytest.fixture
def sample_data_provider(sample_admin_areas):
    """DataProvider pre-loaded with test data."""
    provider = DataProvider(api_client=MagicMock())
    provider.loaded_data[DataSource.ADMIN_AREAS] = LoadedDataSource(
        data_type=DataType.ADMIN_AREA_SET,
        data_source=DataSource.ADMIN_AREAS,
        data=sample_admin_areas,
    )
    return provider
```

## Running Tests

```bash
# Unit tests only (fast, no external deps)
uv run pytest tests/unit/

# Infrastructure integration tests
uv run pytest tests/integration_infra/

# Everything with coverage
uv run pytest --cov=my_pipeline --cov-report=term-missing

# Skip slow tests
uv run pytest -m "not integration"

# Skip tests that need API keys (for CI without secrets)
uv run pytest -m "not needs_secrets"
```

## `needs_secrets` Marker for Secret-Dependent Tests

Tests that talk to real external APIs need secrets (API keys, storage credentials). Mark them so CI can skip them:

```python
# conftest.py
import pytest
from dotenv import load_dotenv

@pytest.fixture(autouse=True)
def load_env_for_secrets(request):
    """Load secrets from .env when marker needs_secrets is set."""
    if "needs_secrets" in request.keywords:
        load_dotenv()
```

```python
# tests/integration/test_ipc_integration.py
@pytest.mark.needs_secrets
def test_ipc_extractor_integration(tmp_path):
    storage = LocalStorage()
    extractor = IpcExtractor("IPC", storage, tmp_path, CONFIGS["SOM"].ipc_data_request)
    asyncio.run(extractor.extract())
    assert len(os.listdir(tmp_path)) == 2

# tests/unit/test_dtm.py  — no marker, runs everywhere
def test_select_by_operation(dtm_extractor):
    ...  # uses MagicMock storage, no real API calls
```

Register the marker in `pyproject.toml`:

```toml
[tool.pytest.ini_options]
markers = [
    "needs_secrets: marks tests that require API keys or storage credentials",
    "integration: marks integration tests",
]
```

### CI configuration

```yaml
# In CI, run only tests that don't need secrets
- run: uv run pytest tests/ -m "not needs_secrets" --cov=my_pipeline
```

## Capturing Log Output in Tests

When using loguru, capture log messages for assertion:

```python
from loguru._handler import Message

@pytest.fixture
def loguru_messages():
    """Capture loguru messages for assertion in tests."""
    messages: list[Message] = []
    def sink(message: Message) -> None:
        messages.append(message)
    sink_id = logger.add(sink)
    yield messages
    logger.remove(sink_id)

def test_extract_logs_failure(loguru_messages):
    extractors = [DummyExtractor(), FailingDummyExtractor()]
    pipeline = ETLPipeline(extractors=extractors)
    with pytest.raises(ExceptionGroup):
        pipeline.extract()
    assert any("failed" in str(m) for m in loguru_messages)
```
