# 02 — App Configuration

## App Factory Pattern

Use an **[application factory](README.md#glossary)** (`create_app()`) instead of module-level `app = FastAPI()`. This enables:
- Injecting fakes/mocks for testing without monkeypatching
- Running multiple app instances with different configs
- Clean lifespan management

```python
# api/app.py
from fastapi import FastAPI

def create_app(*, llm_factory: LLMFactory | None = None) -> FastAPI:
    """Application factory — composition root."""
    app = FastAPI(
        title="My App",
        version=__version__,
        lifespan=_make_lifespan(llm_factory or default_llm_factory),
    )

    # Middleware (order matters — outermost first)
    app.add_middleware(RequestLoggingMiddleware)
    app.add_middleware(RequestIdMiddleware)

    # Routes
    app.include_router(router, prefix="/v1")

    # Exception handlers
    register_exception_handlers(app)

    return app
```

```python
# main.py — entry point
import uvicorn
from api.app import create_app

app = create_app()

if __name__ == "__main__":
    uvicorn.run("main:app", host="0.0.0.0", port=8000)
```

## When a Simple `app = FastAPI()` Is Fine

For small services (≤5 endpoints, no complex DI), a module-level app is acceptable:

```python
# main.py
from fastapi import FastAPI
from fastapi.responses import RedirectResponse
from routes.chat import router as chat_router
from routes.search import router as search_router

app = FastAPI(
    title="my-app",
    description="Short description.",
    version="0.1.0",
    license_info={"name": "Apache-2.0", "url": "https://www.apache.org/licenses/LICENSE-2.0"},
)

@app.get("/", include_in_schema=False)
async def docs_redirect():
    return RedirectResponse(url="/docs")

app.include_router(chat_router)
app.include_router(search_router)

if __name__ == "__main__":
    import uvicorn
    uvicorn.run("main:app", host="0.0.0.0", port=int(os.environ.get("PORT", 8000)))
```

## Lifespan Events

Use the **[lifespan](README.md#glossary)** context manager (not deprecated `@app.on_event`):

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: build expensive resources
    settings = AppSettings()
    app.state.db = await create_engine(settings.db.url)
    app.state.orchestrator = build_orchestrator(settings)
    yield
    # Shutdown: cleanup
    await app.state.db.dispose()
```

## Middleware

**[Middleware](README.md#glossary)** wraps every request/response. Order matters — outermost middleware is added last.

### Request ID Middleware

Every request gets a unique ID for tracing through logs:

```python
import secrets
from starlette.types import ASGIApp, Receive, Scope, Send

class RequestIdMiddleware:
    def __init__(self, app: ASGIApp):
        self.app = app

    async def __call__(self, scope: Scope, receive: Receive, send: Send):
        if scope["type"] != "http":
            return await self.app(scope, receive, send)

        request_id = f"req_{secrets.token_urlsafe(16)}"
        # Ensure scope["state"] exists (may not in all ASGI servers)
        scope.setdefault("state", {})
        scope["state"]["request_id"] = request_id

        async def send_with_id(message):
            if message["type"] == "http.response.start":
                # ASGI headers are a list of [name, value] byte-pairs
                headers = list(message.get("headers", []))
                headers.append((b"x-request-id", request_id.encode()))
                message["headers"] = headers
            await send(message)

        await self.app(scope, receive, send_with_id)
```

### Request Logging Middleware

Log every request's method, path, status, and duration (never log bodies or secrets):

```python
import time
import logging
from starlette.types import ASGIApp, Receive, Scope, Send

logger = logging.getLogger(__name__)

class RequestLoggingMiddleware:
    def __init__(self, app: ASGIApp):
        self.app = app

    async def __call__(self, scope: Scope, receive: Receive, send: Send):
        if scope["type"] != "http":
            return await self.app(scope, receive, send)

        start = time.perf_counter()
        status_code = 500  # default in case of unhandled error

        async def send_wrapper(message):
            nonlocal status_code
            if message["type"] == "http.response.start":
                status_code = message["status"]
            await send(message)

        await self.app(scope, receive, send_wrapper)

        elapsed = time.perf_counter() - start
        logger.info(
            "request",
            extra={
                "method": scope["method"],
                "path": scope["path"],
                "status": status_code,
                "duration_ms": round(elapsed * 1000),
                "request_id": scope.get("state", {}).get("request_id"),
            },
        )
```

## OpenAPI Metadata

Always include:

```python
app = FastAPI(
    title="my-app",
    description="What it does. Built by [NLRC 510](https://www.510.global/).",
    version="0.1.0",
    license_info={"name": "Apache-2.0", "url": "https://www.apache.org/licenses/LICENSE-2.0"},
    openapi_tags=[
        {"name": "chat", "description": "Chat endpoints"},
        {"name": "search", "description": "Search endpoints"},
    ],
)
```