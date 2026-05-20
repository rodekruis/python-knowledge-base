# 07 — Testing

## Test Pyramid

Organize tests in three tiers:

```
tests/
├── conftest.py              # shared fixtures, env isolation
├── domain/                  # unit tests (fast, no I/O)
│   ├── test_models.py
│   └── test_errors.py
├── services/                # unit tests with mocked ports
│   └── test_orchestrator.py
├── api/                     # route tests with TestClient
│   ├── conftest.py          # app fixture, fake dependencies
│   ├── test_routes.py
│   └── test_auth.py
├── adapters/                # adapter tests (may mock HTTP)
│   └── test_db.py
├── integration/             # needs real DB/services
│   └── conftest.py
└── e2e/                     # full app startup
    └── conftest.py
```

| Tier | Speed | Dependencies | Marker |
|------|-------|-------------|--------|
| Unit | ⚡ fast | None | (default) |
| Integration | 🐢 slower | Postgres, Redis | `@pytest.mark.integration` |
| E2E | 🐌 slowest | Full app + externals | `@pytest.mark.e2e` |

## Fixtures and Configuration

### Environment isolation

Prevent tests from inheriting your local `.env`:

```python
# tests/conftest.py
import os
import pytest

@pytest.fixture(autouse=True)
def _isolate_env(monkeypatch):
    """Strip all app-related env vars so tests are deterministic."""
    prefixes = ("LLM_", "DB_", "AUTH_", "OPENAI_", "VECTOR_STORE_")
    for key in list(os.environ):
        if key.startswith(prefixes):
            monkeypatch.delenv(key, raising=False)
```

### Minimal env vars for tests

Set only what's needed for module imports:

```python
# tests/conftest.py
import sys
from unittest.mock import MagicMock

# Mock heavy modules BEFORE they're imported
sys.modules.setdefault("agents.rag_agent", MagicMock(get_rag_agent=MagicMock()))

# Set minimal env vars required for import
os.environ.setdefault("PORT", "8000")
os.environ.setdefault("API_KEY", "test-key")

import pytest
from fastapi.testclient import TestClient
from main import app

@pytest.fixture
def client():
    return TestClient(app)
```

## Route Testing Patterns

### Simple pattern (TestClient)

For smaller apps:

```python
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

class TestSearch:
    @patch("routes.search.get_vector_store")
    def test_basic_search(self, mock_vs, client):
        mock_vs.return_value.similarity_search_with_score.return_value = [
            (MagicMock(metadata={"question": "Q", "answer": "A"}), 0.9)
        ]
        resp = client.post("/search", json={"query": "hello", "googleSheetId": "abc"})
        assert resp.status_code == 200
        assert len(resp.json()["results"]) > 0
```

### App factory pattern (QFA-style)

For larger apps — create a fresh app per test with injected fakes:

```python
# tests/api/conftest.py
import pytest
from fastapi.testclient import TestClient
from api.app import create_app

class FakeOrchestrator:
    async def analyze(self, request):
        return AnalysisResult(summary="test result")

@pytest.fixture
def fake_orchestrator():
    return FakeOrchestrator()

@pytest.fixture
def client(fake_orchestrator):
    app = create_app(llm_factory=lambda _: FakeLLM())
    app.state.orchestrator = fake_orchestrator
    app.state.api_keys = [TenantApiKey(tenant_id="test", key=SecretStr("test-key"))]
    return TestClient(app)
```

## Mocking External Services

### Patch at the route module level

```python
# The import path must match WHERE it's used, not where it's defined
@patch("routes.chat.get_rag_agent")
def test_chat(mock_get_agent, client):
    mock_agent = MagicMock()
    mock_agent.invoke.return_value = {"messages": [MagicMock(content="Hello")]}
    mock_get_agent.return_value = mock_agent

    resp = client.post("/chat-dummy", json={"message": "hi"})
    assert resp.status_code == 200
```

### Never use `test_mode` flags in production code

❌ Bad:
```python
@router.post("/kobo-to-121")
async def kobo_to_121(request: Request, test_mode: bool = False):
    if test_mode:
        return {"payload": prepared_data}  # skip real API call
    ...
```

✅ Good — inject the dependency:
```python
@router.post("/kobo-to-121")
async def kobo_to_121(request: Request, client_121=Depends(get_121_client)):
    result = await client_121.send(prepared_data)
    ...

# In tests: override the dependency
app.dependency_overrides[get_121_client] = lambda: FakeClient()
```

> **Tip:** [`dependency_overrides`](README.md#glossary) is the idiomatic way to swap real dependencies for fakes in FastAPI tests — no monkeypatching needed.

## What to Test

| Layer | What to test | How |
|-------|-------------|-----|
| Routes | Status codes, response shape, auth rejection | TestClient + mocked deps |
| Business logic | Core transformations, edge cases | Direct function calls |
| Adapters | Correct API calls, error mapping | Mock HTTP (responses, httpx_mock) |
| Integration | DB queries, migrations | Real Postgres (docker-compose) |

## Running Tests

```bash
# All unit tests (fast)
uv run pytest tests/ -v --tb=short

# Exclude integration tests (default for quick feedback)
uv run pytest tests/ -v --tb=short -m "not integration"

# Only integration
uv run pytest tests/ -m integration

# With coverage
uv run pytest tests/ --cov=src --cov-report=term-missing
```