# 11 — Logging and Observability

> **Shared foundation:** See [../shared/logging.md](../shared/logging.md) for the universal logging setup, log levels, what to/not to log, structured logging with `extra`, Azure Monitor OpenTelemetry, and health check patterns.

This addendum covers **Flask-specific** logging integration only.

---

## Flask Logging Setup

Configure logging in the application factory:

```python
# src/my_app/logging_config.py
import logging
import sys


def configure_logging(app):
    """Configure structured logging for the Flask app."""
    log_level = logging.DEBUG if app.debug else logging.INFO

    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(logging.Formatter(
        "%(asctime)s | %(levelname)-8s | %(name)s | %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S",
    ))

    root = logging.getLogger()
    root.setLevel(log_level)
    root.addHandler(handler)

    # Silence noisy libraries
    logging.getLogger("urllib3").setLevel(logging.WARNING)
    logging.getLogger("werkzeug").setLevel(logging.WARNING)
```

```python
# src/my_app/__init__.py
def create_app(config_class=Config):
    app = Flask(__name__)
    app.config.from_object(config_class)
    configure_logging(app)
    return app
```

---

## Request Logging with Hooks

Flask uses `@before_request` / `@after_request` for per-request logging:

```python
import time
from flask import g, request

@app.before_request
def start_timer():
    g.start_time = time.perf_counter()

@app.after_request
def log_request(response):
    duration = time.perf_counter() - g.start_time
    logger.info(
        "Request completed",
        extra={
            "method": request.method,
            "path": request.path,
            "status": response.status_code,
            "duration_ms": round(duration * 1000, 1),
            "remote_addr": request.remote_addr,
        },
    )
    return response
```

> **Tip:** Exclude `/health` from logging by adding an early return:
> ```python
> @app.after_request
> def log_request(response):
>     if request.path == "/health":
>         return response
>     # ... rest of logging
> ```

---

## Azure Application Insights (Legacy)

Existing Flask apps may use the opencensus integration:

```python
from opencensus.ext.azure.log_exporter import AzureLogHandler
from opencensus.ext.flask.flask_middleware import FlaskMiddleware
from opencensus.ext.azure.trace_exporter import AzureExporter
from opencensus.trace.samplers import ProbabilitySampler


def configure_observability(app):
    conn_str = os.environ.get("APPLICATIONINSIGHTS_CONNECTION_STRING")
    if not conn_str:
        return

    FlaskMiddleware(
        app,
        exporter=AzureExporter(connection_string=conn_str),
        sampler=ProbabilitySampler(rate=1.0),
    )
    logging.getLogger().addHandler(AzureLogHandler(connection_string=conn_str))
```

> **Migration:** `opencensus` is deprecated. New projects should use `azure-monitor-opentelemetry` — see [../shared/logging.md](../shared/logging.md#azure-monitor-opentelemetry).

---

## Health Check

```python
@main_bp.route("/health")
def health():
    return {"status": "ok"}, 200
```

Configure Azure App Service health probe to poll this endpoint.

