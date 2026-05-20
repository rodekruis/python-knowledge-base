# 01 — Project Structure

## Recommended Layout

For anything beyond a single-endpoint microservice, use a proper package structure:

```
my-app/
├── src/my_app/              # source root
│   ├── __init__.py          # version via importlib.metadata
│   ├── main.py              # uvicorn entry point
│   ├── settings.py          # pydantic-settings configuration
│   ├── domain/              # business logic (pure Python, no framework deps)
│   │   ├── models.py        # domain models (frozen Pydantic or dataclasses)
│   │   ├── errors.py        # domain exception hierarchy
│   │   └── ports.py         # Protocol-based interfaces (ABC/Protocol)
│   ├── services/            # use-case orchestration
│   │   └── orchestrator.py  # coordinates domain + adapters
│   ├── adapters/            # external system integrations
│   │   ├── llm_client.py    # LLM provider adapter
│   │   ├── db.py            # database repository
│   │   └── ...
│   └── api/                 # FastAPI-specific code
│       ├── app.py           # create_app() factory
│       ├── routes.py        # route handlers
│       ├── schemas.py       # API request/response models
│       └── dependencies.py  # FastAPI Depends
├── tests/
│   ├── conftest.py
│   ├── domain/
│   ├── services/
│   ├── adapters/
│   ├── api/
│   ├── integration/
│   └── e2e/
├── alembic/                 # DB migrations (if applicable)
├── infra/                   # Terraform/IaC
├── docs/                    # ADRs, architecture docs
├── Dockerfile
├── entrypoint.sh
├── pyproject.toml
├── uv.lock
├── Makefile
└── README.md
```

## When a Flat Structure Is OK

For true microservices with ≤3 endpoints and no complex business logic:

```
my-app/
├── main.py
├── routes/
│   └── my_routes.py
├── utils/
│   ├── logger.py
│   └── config.py
├── tests/
├── Dockerfile
├── pyproject.toml
└── uv.lock
```

## Key Principles

### Separate API schemas from domain models

Keep **[API schemas and domain models](README.md#glossary)** separate. API request/response models live in `api/schemas.py`. Domain models live in `domain/models.py`. Route handlers explicitly map between them. This lets the API contract evolve independently of internal representations.

```python
# api/schemas.py — what the HTTP client sees
class ApiAnalyzeRequest(BaseModel):
    documents: list[str]
    language: str = "en"

# domain/models.py — internal representation
class AnalysisRequest:
    documents: tuple[str, ...]
    language: Language
```

### Layer boundaries via imports

Enforce that:
- `domain/` imports nothing from the project (pure business logic)
- `services/` imports only from `domain/`
- `api/` imports from `domain/` and `services/`
- **[`adapters/`](README.md#glossary)** imports only from `domain/` (implements ports)

This can be enforced in CI with [`import-linter`](https://github.com/seddonym/import-linter).

### No module-level singletons for external resources

❌ Bad — untestable, fails at import time if env vars are missing:
```python
# agents/rag_agent.py
llm = AzureChatOpenAI(...)  # runs at import
agent = StateGraph(...).compile()  # module-level singleton
```

✅ Good — lazy init or factory function:
```python
def get_rag_agent() -> CompiledStateGraph:
    """Build or return the cached agent."""
    ...
```

✅ Best — injected via app factory:
```python
def create_app(*, llm_factory=None) -> FastAPI:
    app = FastAPI(lifespan=_make_lifespan(llm_factory))
    ...
```
