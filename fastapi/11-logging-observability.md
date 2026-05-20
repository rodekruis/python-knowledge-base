# 11 — Logging and Observability

> **Shared foundation:** See [../shared/logging.md](../shared/logging.md) for the universal logging setup, log levels, what to/not to log, structured logging with `extra`, Azure Monitor OpenTelemetry, and health check patterns.

This addendum covers **FastAPI-specific** logging integration only.

---

## FastAPI Logging Setup

Configure logging at application startup using the lifespan:

```python
# src/my_app/log.py
import logging
import sys
import os


def setup_logging() -> logging.Logger:
    """Configure structured logging. Call once at app startup."""
    level = logging.DEBUG if os.getenv("ENV") == "development" else logging.INFO

    logging.basicConfig(
        level=level,
        format="%(asctime)s | %(levelname)-8s | %(name)s | %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S",
        stream=sys.stdout,
        force=True,
    )

    # Silence noisy libraries
    for name in ("httpx", "azure", "openai", "uvicorn.access"):
        logging.getLogger(name).setLevel(logging.WARNING)

    return logging.getLogger(__name__)
```

```python
# src/my_app/main.py
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    setup_logging()
    yield

app = FastAPI(lifespan=lifespan)
```

---

## Request Logging with ASGI Middleware

FastAPI uses middleware for per-request logging:

```python
import time
import logging
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response

logger = logging.getLogger(__name__)


class RequestLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next) -> Response:
        if request.url.path in ("/health", "/ready"):
            return await call_next(request)

        start = time.perf_counter()
        response = await call_next(request)
        duration_ms = round((time.perf_counter() - start) * 1000, 1)

        logger.info(
            "%s %s → %s (%sms)",
            request.method,
            request.url.path,
            response.status_code,
            duration_ms,
            extra={
                "method": request.method,
                "path": request.url.path,
                "status": response.status_code,
                "duration_ms": duration_ms,
            },
        )
        return response
```

```python
app.add_middleware(RequestLoggingMiddleware)
```

---

## Azure Application Insights (OpenTelemetry)

FastAPI projects use the OpenTelemetry SDK directly:

```python
import os
from azure.monitor.opentelemetry import configure_azure_monitor


def setup_azure_monitor() -> None:
    conn_str = os.environ.get("APPLICATIONINSIGHTS_CONNECTION_STRING")
    if conn_str:
        configure_azure_monitor()
```

Call in lifespan before `yield`. This automatically traces async requests, logs, and metrics.

---

## Request-Level Context

Include correlation IDs for traceability:

```python
import uuid
from starlette.middleware.base import BaseHTTPMiddleware

class RequestIdMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        request.state.request_id = str(uuid.uuid4())
        response = await call_next(request)
        response.headers["X-Request-ID"] = request.state.request_id
        return response
```

Use `request.state.request_id` in log `extra` fields for end-to-end tracing.

