# FastAPI Best Practices

Best practices for building FastAPI-based applications. Based on industry-standard FastAPI patterns and our own work.

## 🚀 Starting a new project

Copy [copilot-instructions.md](copilot-instructions.md) into your repo at `.github/copilot-instructions.md`. It consolidates the most important rules from this guide (and the [shared](../shared/) standards) into a single concise file that GitHub Copilot reads automatically.

## Guide Structure

| File | Topic |
|------|-------|
| [01-project-structure.md](01-project-structure.md) | Directory layout, packaging, and imports |
| [02-app-configuration.md](02-app-configuration.md) | FastAPI app setup, middleware, lifespan |
| [03-routes-and-schemas.md](03-routes-and-schemas.md) | Routers, dependency injection, Pydantic models |
| [04-configuration-management.md](04-configuration-management.md) | Environment variables, settings, secrets |
| [05-error-handling.md](05-error-handling.md) | Exception hierarchy, global handlers, error responses |
| [06-authentication.md](06-authentication.md) | API keys, Bearer tokens, multi-tenant auth |
| [07-testing.md](07-testing.md) | Test pyramid, fixtures, mocking, CI |
| [08-docker.md](08-docker.md) | FastAPI Dockerfile, ASGI entrypoint (→ [shared/docker](../shared/docker.md)) |
| [09-ci-cd.md](09-ci-cd.md) | Docker deploy to Azure (→ [shared/ci-cd](../shared/ci-cd.md)) |
| [10-dependencies.md](10-dependencies.md) | FastAPI-specific deps (→ [shared/dependencies](../shared/dependencies.md)) |
| [11-logging-observability.md](11-logging-observability.md) | ASGI middleware logging (→ [shared/logging](../shared/logging.md)) |
| [12-code-quality.md](12-code-quality.md) | FastAPI type patterns (→ [shared/code-quality](../shared/code-quality.md)) |

## Glossary

FastAPI-specific terms used throughout these guides. For shared terms (uv, Ruff, Gunicorn, Lockfile, etc.) see the [shared glossary](../shared/README.md#glossary).

| Term | Definition |
|------|-----------|
| **Application factory** | A function (conventionally `create_app()`) that constructs and returns a configured `FastAPI` instance. Enables injecting different configurations for testing, development, and production without relying on global state. See [02-app-configuration.md](02-app-configuration.md). |
| **ASGI (Asynchronous Server Gateway Interface)** | The Python standard for async web servers and applications. FastAPI is an ASGI application; Uvicorn is an ASGI server. The async counterpart of WSGI. See [08-docker.md](08-docker.md). |
| **Dependency injection (`Depends`)** | FastAPI's mechanism for declaring that a route handler needs something (auth, database, settings, etc.) provided to it. FastAPI resolves the dependency tree automatically, enabling testability and separation of concerns. See [03-routes-and-schemas.md](03-routes-and-schemas.md). |
| **Lifespan** | An `asynccontextmanager` passed to `FastAPI(lifespan=...)` that defines startup logic (before `yield`) and shutdown logic (after `yield`). Replaces the deprecated `@app.on_event("startup")` pattern. See [02-app-configuration.md](02-app-configuration.md). |
| **Middleware** | Code that wraps every request/response cycle. In FastAPI/Starlette, middleware can be raw ASGI classes or `BaseHTTPMiddleware` subclasses. Used for logging, request IDs, CORS, etc. See [02-app-configuration.md](02-app-configuration.md). |
| **Pydantic model / schema** | A class inheriting from `pydantic.BaseModel` used to declare, validate, and serialize data. In FastAPI, request bodies and response shapes are defined as Pydantic models, which also drive the auto-generated OpenAPI docs. See [03-routes-and-schemas.md](03-routes-and-schemas.md). |
| **pydantic-settings (`BaseSettings`)** | A Pydantic extension that reads configuration from environment variables (and `.env` files) with type validation and `SecretStr` support. See [04-configuration-management.md](04-configuration-management.md). |
| **Router (`APIRouter`)** | A FastAPI object that groups related endpoints, similar to Flask's Blueprint. Routers are included in the app with `app.include_router(...)` and support tags and prefixes for OpenAPI organization. See [03-routes-and-schemas.md](03-routes-and-schemas.md). |
| **`SecretStr`** | A Pydantic type that masks the value in `repr()` / logging output (`'**********'`). Prevents accidental exposure of API keys and passwords. See [04-configuration-management.md](04-configuration-management.md). |
| **Uvicorn** | A high-performance ASGI server that runs FastAPI apps. Used directly in development (`uvicorn --reload`) and behind Gunicorn workers in production. See [08-docker.md](08-docker.md). |
| **Gunicorn + UvicornWorker** | The production deployment pattern: Gunicorn manages multiple Uvicorn worker processes for concurrency and process resilience. See [08-docker.md](08-docker.md). |
| **`dependency_overrides`** | A `dict` on the `FastAPI` app that lets you swap out real dependencies for fakes/mocks during testing — without monkeypatching. See [07-testing.md](07-testing.md). |
| **Domain model vs API schema** | Domain models represent internal business logic (pure Python, no HTTP concepts). API schemas represent the external HTTP contract (request/response shapes). Keeping them separate allows the API to evolve independently of internals. See [01-project-structure.md](01-project-structure.md). |
| **Adapter** | A module that wraps an external system (LLM provider, database, third-party API) behind an interface defined in `domain/ports.py`. Adapters are the outermost layer and can be swapped for fakes in testing. See [01-project-structure.md](01-project-structure.md). |
| **Hexagonal architecture (ports & adapters)** | The architectural style underpinning this layout: a pure `domain/` core surrounded by `adapters/` that plug into it, with all dependencies pointing *inward* toward the domain (Dependency Inversion). Yields testability, replaceable infrastructure, and a framework-free core. See [01-project-structure.md](01-project-structure.md#key-principles). |
| **Port** | A `Protocol`/ABC interface declared in `domain/ports.py`, phrased in domain terms, describing something the core needs from the outside world (a store, an LLM, a queue). Services depend on ports; adapters implement them. See [01-project-structure.md](01-project-structure.md#key-principles). |
| **Composition root** | The single outermost place — typically `create_app()` / `lifespan` in `api/app.py` — where concrete adapters are constructed and wired into the ports that services expect. The only layer allowed to know about everything. See [02-app-configuration.md](02-app-configuration.md). |
