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

These principles all flow from one idea: **[hexagonal architecture](https://alistair.cockburn.us/hexagonal-architecture/)** (a.k.a. "ports and adapters"). Picture the app as a hexagon. In the centre sits your **domain** - the business rules, written in plain Python that knows nothing about HTTP, databases, or LLM SDKs. Everything the domain needs from the outside world (a database, an LLM, a search index) it describes as a **port**: an interface (`Protocol`/ABC) phrased in domain terms. The concrete implementations - the Azure client, the Postgres repository, the FastAPI layer - are **adapters** that plug into those ports from the outside.

The crucial rule is the **direction of dependencies**: everything points *inward*, toward the domain. The domain never imports an adapter; adapters import the domain. This is the **Dependency Inversion Principle** applied at the architecture level — high-level policy (business logic) doesn't depend on low-level detail (infrastructure); both depend on the abstraction (the port).

Why bother? Three concrete payoffs:

- **Testability.** Because services talk to ports, not concrete clients, a test can pass in a fake adapter (an in-memory store, a stub LLM) without spinning up Azure or a database. Your business logic is testable in milliseconds.
- **Replaceability.** Swapping Azure AI Search for Postgres+pgvector, or one LLM provider for another, means writing a new adapter and changing one line in the composition root. The domain and services don't change at all.
- **Comprehensibility.** When the core has zero framework imports, you can read and reason about *what the app does* without wading through *how it talks to the outside world*. The two concerns stay separate.

The rest of these principles are just this idea made concrete.

### Separate API schemas from domain models

Keep **[API schemas and domain models](README.md#glossary)** separate. API request/response models live in `api/schemas.py`. Domain models live in `domain/models.py`. Route handlers explicitly map between them. This lets the API contract evolve independently of internal representations.

**Why, in hexagonal terms:** the HTTP layer is an *adapter* — one of potentially many ways into the application (today REST, tomorrow a CLI, a queue consumer, or a gRPC endpoint). A Pydantic request body is part of the **HTTP contract**, shaped by what's convenient to send over the wire and what you've promised external clients. A domain model is part of the **core**, shaped by the business rules. If you let the domain reuse the API's Pydantic models, you've inverted the dependency — the core now depends on a detail of one particular adapter, and you can no longer change the wire format without touching business logic (or vice versa). The explicit mapping in the handler is the seam that keeps the transport swappable.

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

**Why, in hexagonal terms:** these four rules *are* the inward-pointing dependency arrows of the hexagon, written as import constraints a tool can check.

- `domain/` imports nothing from the project because it's the centre — if it reached out to `services`, `adapters`, or `api`, those details would leak into the core and the whole scheme collapses. A domain that imports nothing is one you can lift out and reuse, and one whose tests need no infrastructure.
- `services/` import only `domain/` (the models *and the ports*), never a concrete adapter. A service says "give me something that satisfies the `VectorStore` port" — it neither knows nor cares that the real thing is Azure. That's what lets you inject a fake in tests and swap implementations in production.
- `adapters/` import only `domain/` because their whole job is to *implement* a domain port (and translate the outside world — an SDK exception, a JSON row — into domain terms). An adapter that imported `services` or `api` would be reaching back inward, coupling infrastructure to use-cases.
- `api/` may import both `domain` and `services` because it's the outermost adapter plus the **composition root**: the one place allowed to know about everything, where you wire concrete adapters into the ports the services expect (typically in `create_app()` / `lifespan`).

Because these are plain import rules, they don't enforce themselves — a moment's inattention reintroduces the coupling. So pin them down in CI with [`import-linter`](https://github.com/seddonym/import-linter) (a `layers` contract for the ordering plus `forbidden` contracts for "domain imports nothing" and "services never import adapters"). The linter turns an architectural intention into a build that fails loudly when someone crosses a boundary.

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

**Why, in hexagonal terms:** a module-level `llm = AzureChatOpenAI(...)` is a concrete adapter instantiating itself, eagerly, the instant its module is imported. That defeats the whole point of ports-and-adapters. The composition root no longer decides which adapter is used — the import does — so a test can't substitute a fake without real credentials and network access, and importing the module for *any* reason (even to read one constant) triggers construction and fails fast if an env var is missing. Pushing creation into a factory or the `lifespan` keeps the **choice of adapter** where it belongs: at the outermost edge, injected inward, so the same core can run against the real Azure client in production and a stub in tests.

