# 07 — Testing

## Test Setup

### Application factory makes testing easy

```python
# tests/conftest.py
import pytest
from my_app import create_app
from my_app.config import TestConfig


@pytest.fixture
def app():
    """Create application for testing."""
    app = create_app(TestConfig)
    yield app


@pytest.fixture
def client(app):
    """A test client for the app."""
    return app.test_client()


@pytest.fixture
def runner(app):
    """A test CLI runner for the app."""
    return app.test_cli_runner()
```

### TestConfig

```python
class TestConfig(Config):
    TESTING = True
    DEBUG = True
    SECRET_KEY = "test-secret-key"
    SESSION_COOKIE_SECURE = False
    WTF_CSRF_ENABLED = False       # Disable CSRF in tests for convenience
```

## Test Pyramid

```
tests/
├── conftest.py              # shared fixtures
├── test_routes.py           # route / integration tests
├── test_services.py         # unit tests for business logic
├── test_forms.py            # form validation tests
└── test_clients.py          # external API client tests (mocked)
```

| Tier | What | Speed | Dependencies |
|------|------|-------|-------------|
| Unit | Services, utils, pure functions | ⚡ fast | None |
| Form | WTForms validation logic | ⚡ fast | App context |
| Route | HTTP request → response | 🐢 medium | Full app, mocked services |
| Integration | Real external calls | 🐌 slow | Network, mark with `@pytest.mark.integration` |

## Route Testing

### GET request

```python
def test_index_get(client):
    response = client.get("/")
    assert response.status_code == 200
    assert b"Batch Creation" in response.data
```

### POST with form data

```python
def test_index_post_success(client, mocker):
    # Mock the service layer
    mock_result = mocker.MagicMock()
    mock_result.batch_names = ["CLB-001", "CLB-002"]
    mocker.patch("my_app.main.routes.batch_service.create_and_sample", return_value=mock_result)

    response = client.post("/", data={
        "username": "test@example.com",
        "password": "testpass",
        "program_id": 42,
        "gn_label": "Label1",
        "dsdiv": "DSD1",
        "district": "District1",
        "batch_size": 30,
        "min_batchsize": "yes",
    }, follow_redirects=True)

    assert response.status_code == 200
    assert b"CLB-001" in response.data
```

### Testing error states

```python
def test_index_post_auth_failure(client, mocker):
    mocker.patch(
        "my_app.main.routes.batch_service.create_and_sample",
        side_effect=AuthenticationError("Invalid credentials"),
    )

    response = client.post("/", data={...}, follow_redirects=True)
    assert b"Invalid" in response.data
```

### Testing with session data

```python
def test_success_page_shows_batch_names(client):
    with client.session_transaction() as sess:
        sess["batch_names"] = ["CLB-001", "CLB-002"]
        sess["program_id"] = 42

    response = client.get("/success")
    assert response.status_code == 200
    assert b"CLB-001" in response.data
```

## Service / Unit Testing

Business logic functions should be testable **without** Flask:

```python
# tests/test_services.py
import pytest
from my_app.services.batch_service import create_batches
import pandas as pd


def test_create_batches_correctly_splits():
    df = pd.DataFrame({"referenceId": range(58)})
    batches = create_batches(df, batch_size=30, allow_smaller_last=True)
    assert len(batches) == 2
    assert len(batches[0]) == 30
    assert len(batches[1]) == 28


def test_create_batches_drops_small_last_batch():
    df = pd.DataFrame({"referenceId": range(58)})
    batches = create_batches(df, batch_size=30, allow_smaller_last=False)
    assert len(batches) == 1
    assert len(batches[0]) == 30
```

## Mocking External API Calls

Use `responses` or `pytest-mock` to mock `requests`:

```python
# tests/test_clients.py
import responses
from my_app.clients.api_121 import login_121


@responses.activate
def test_login_success():
    responses.add(
        responses.POST,
        "https://training.121.global/api/users/login",
        json={"access_token_general": "fake-token"},
        status=200,
    )

    token = login_121("user@test.com", "pass", base_url="https://training.121.global/api")
    assert token == "fake-token"


@responses.activate
def test_login_failure_raises():
    responses.add(
        responses.POST,
        "https://training.121.global/api/users/login",
        status=401,
    )

    with pytest.raises(AuthenticationError):
        login_121("user@test.com", "wrong", base_url="https://training.121.global/api")
```

## Form Validation Testing

```python
# tests/test_forms.py
from my_app.main.forms import BatchForm


def test_valid_form(app):
    with app.test_request_context():
        form = BatchForm(data={
            "username": "user@test.com",
            "password": "pass",
            "program_id": 42,
            "gn_label": "Label1",
            "dsdiv": "DSD1",
            "district": "District1",
            "batch_size": 30,
            "min_batchsize": "yes",
        })
        assert form.validate()


def test_missing_required_field(app):
    with app.test_request_context():
        form = BatchForm(data={"username": "user@test.com"})
        assert not form.validate()
        assert "password" in form.errors
```

## Running Tests

```bash
# Run all tests
uv run pytest

# Run with coverage
uv run pytest --cov=my_app --cov-report=term-missing

# Run only unit tests (skip integration tests)
uv run pytest -m "not integration"
```

## Coverage Target

Aim for:
- **90%+** on business logic / services
- **80%+** overall
- **100%** on security-critical code (auth, permissions)
