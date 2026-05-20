# Logging & Observability

## Core Principles

1. **One logger per module** — `logger = logging.getLogger(__name__)`
2. **Structured data via `extra`** — not string interpolation
3. **Never log secrets** — tokens, passwords, API keys, PII
4. **Log at the right level** — don't spam INFO with noise

---

## Standard Setup

```python
# src/my_app/log.py
import logging
import sys


def setup_logging(level: str = "INFO") -> None:
    """Configure logging for the application.

    Call once at startup (in create_app / lifespan / main).
    """
    logging.basicConfig(
        level=getattr(logging, level.upper(), logging.INFO),
        format="%(asctime)s | %(levelname)-8s | %(name)s | %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S",
        stream=sys.stdout,
        force=True,
    )
    # Silence noisy libraries
    logging.getLogger("httpx").setLevel(logging.WARNING)
    logging.getLogger("azure").setLevel(logging.WARNING)
    logging.getLogger("uvicorn.access").setLevel(logging.WARNING)
```

### Usage in Modules

```python
import logging

logger = logging.getLogger(__name__)

def process_item(item_id: str) -> None:
    logger.info("Processing item", extra={"item_id": item_id})
    # ...
    logger.debug("Validation passed", extra={"item_id": item_id})
```

---

## Log Levels

| Level | Use for | Example |
|-------|---------|---------|
| `DEBUG` | Detailed diagnostic info (off in production) | Variable values, query params |
| `INFO` | Normal operations worth recording | Request handled, job started/completed |
| `WARNING` | Unexpected but recoverable situations | Retry attempt, deprecated usage |
| `ERROR` | Failures that need attention | API call failed, invalid data |
| `CRITICAL` | System-wide failures | Database unreachable, out of memory |

### Production Default: `INFO`

Set `DEBUG` only in development. Never log at DEBUG level in production — it generates enormous volume and may expose sensitive data.

---

## What to Log

✅ **Do log:**
- Incoming request method + path (at INFO)
- Response status + duration (at INFO)
- Business events: "user created", "payment processed"
- Errors with enough context to reproduce
- External API calls: URL, status, duration

❌ **Never log:**
- Passwords, tokens, API keys
- Full request/response bodies (may contain PII)
- Credit card numbers, personal identification
- Health check requests (noise)
- Successful auth tokens (security risk)

---

## Structured Logging with `extra`

Prefer structured fields over string formatting:

```python
# ✅ Good — structured, searchable, parseable
logger.info("Order placed", extra={"order_id": order.id, "total": order.total})

# ❌ Bad — unstructured, hard to parse
logger.info(f"Order {order.id} placed, total={order.total}")
```

For JSON output (useful in Azure Monitor / Log Analytics), use a JSON formatter:

```python
import json
import logging


class JsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        log_data = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
        }
        # Include extra fields
        for key in record.__dict__:
            if key not in logging.LogRecord("", 0, "", 0, None, None, None).__dict__:
                log_data[key] = getattr(record, key)
        return json.dumps(log_data)
```

---

## Azure Monitor (OpenTelemetry)

Use the official Azure Monitor OpenTelemetry SDK for production observability:

```bash
uv add azure-monitor-opentelemetry
```

```python
from azure.monitor.opentelemetry import configure_azure_monitor

def setup_azure_monitor() -> None:
    """Configure Azure Application Insights via OpenTelemetry.

    Requires APPLICATIONINSIGHTS_CONNECTION_STRING environment variable.
    """
    configure_azure_monitor()
```

This automatically captures:
- **Traces** — request/response spans
- **Logs** — Python logging output
- **Metrics** — request duration, failure rate

### Environment Variable

```bash
APPLICATIONINSIGHTS_CONNECTION_STRING=InstrumentationKey=xxx;IngestionEndpoint=https://...
```

Set this in Azure App Service → Configuration → Application settings.

> **Migration note:** The older `opencensus-ext-azure` / `opencensus-ext-flask` packages are
> deprecated. New projects must use `azure-monitor-opentelemetry`. Existing projects should
> migrate when possible.

---

## Health Check Endpoint

Every application should expose a health endpoint that monitoring tools can poll:

```python
# Minimal health check (framework-agnostic pattern)
def health_check():
    return {"status": "ok"}
```

- Return `200 OK` with `{"status": "ok"}` when healthy
- Optionally check database connectivity, external services
- **Do not log** health check requests — they generate noise
- Use for Docker HEALTHCHECK, Azure health probes, and uptime monitoring

---

## Request Logging Pattern

Log each request with method, path, status, and duration. Implementation differs by framework
(see framework-specific addenda), but the principle is universal:

```
2024-01-15 10:23:45 | INFO | myapp.middleware | GET /api/users → 200 (45ms)
```

Include:
- HTTP method and path
- Response status code
- Duration in milliseconds
- Correlation/request ID (if available)

Exclude from logging:
- `/health` and `/ready` endpoints
- Static asset requests
- Full request/response bodies
