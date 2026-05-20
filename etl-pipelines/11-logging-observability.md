# 11 — Logging and Observability

> **Shared foundation:** See [../shared/logging.md](../shared/logging.md) for universal principles: logger-per-module, structured `extra={}`, log levels table, what to/not to log, Azure Monitor OpenTelemetry, and health check patterns.

This document covers **pipeline-specific** logging patterns: library choice (loguru vs stdlib), run traceability, duration tracking, and alerting on failure.

---

## Choose a Logging Library

Two good options exist. Pick one per project and commit.

### ❌ Bad: print statements

```python
def run_pipeline():
    print("Starting pipeline...")
    print(f"Loading config from {config_path}")
    try:
        data = extract()
    except Exception as e:
        print(f"ERROR: {e}")
```

### ✅ Option A: stdlib `logging`

```python
import logging

logger = logging.getLogger(__name__)

def run_pipeline():
    logger.info("Starting pipeline", extra={"config": str(config_path)})
    try:
        data = extract()
    except Exception:
        logger.exception("Failed during extraction")
        raise
```

### ✅ Option B: loguru (recommended for new projects)

[loguru](https://github.com/Delgan/loguru) is a single-import replacement for stdlib logging. It's better in every practical way: zero configuration, automatic exception formatting, structured output, and sane defaults.

```python
from loguru import logger

def run_pipeline():
    logger.info("Starting pipeline for {}", config_path)
    try:
        data = extract()
    except Exception:
        logger.exception("Failed during extraction")
        raise
```

Configure once at the entry point:

```python
import sys
from loguru import logger

# Remove default handler, add one with desired level
logger.remove()
logger.add(sys.stderr, level="INFO")

def main():
    config = tyro.cli(RunConfig)
    logger.info("Running pipeline for {}", config.country)
    pipeline = ETLPipeline.from_config(config)
    pipeline.run()
```

Why loguru wins for new projects:

| stdlib `logging` | loguru |
|-------------------|--------|
| Requires `logging.basicConfig()` + formatters | Sensible defaults, zero config |
| `logger = logging.getLogger(__name__)` per module | `from loguru import logger` everywhere |
| `extra={}` dict for structured fields | f-string-style `{}` placeholders |
| Manual JSON formatter class | `logger.add(sink, serialize=True)` |
| Tracebacks need `logger.exception()` | Better traceback formatting built-in |
| Thread-safe but async-unaware | Thread-safe + async-compatible sinks |

## Configure Logging Once, at the Top

Configure logging in the CLI entry point — never inside library code.

```python
# my_pipeline/infra/run_forecasts.py

import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)-8s | %(name)s | %(message)s",
    datefmt="%Y-%m-%dT%H:%M:%S",
)
logger = logging.getLogger(__name__)

@click.command()
def run_forecasts(config, run_target, scenario, issued_at):
    logger.info("Pipeline started", extra={"run_target": run_target})
    ...
```

### Module-scoped loggers

Every module should define its own logger at the top:

```python
# my_pipeline/infra/data_provider.py
import logging

logger = logging.getLogger(__name__)

class DataProvider:
    def get_data(self, data_sources):
        for source in data_sources:
            logger.info("Fetching data source: %s", source.value)
            ...
```

This lets you control log levels per module:

```python
# Silence noisy HTTP libraries
logging.getLogger("urllib3").setLevel(logging.WARNING)
logging.getLogger("httpx").setLevel(logging.WARNING)
```

## What to Log

| Phase | Log What | Level |
|-------|---------|-------|
| Startup | Config file, run target, pipeline type, issued_at | `INFO` |
| Extract | Data source name, record count, duration | `INFO` |
| Transform | Input/output shape, skipped records | `INFO` |
| Load | Destination, record count, status code | `INFO` |
| Validation | Integrity check results | `INFO` / `WARNING` |
| Error | Full exception with traceback | `ERROR` |
| Debug | Raw API responses, intermediate data shapes | `DEBUG` |

### ❌ Bad: log the data

```python
logger.info("Fetched data: %s", json.dumps(data))  # Could be 100 MB
```

### ✅ Good: log metadata about the data

```python
logger.info("Fetched %d records from %s in %.2fs", len(data), source.value, elapsed)
```

## Run Traceability

Every pipeline run should be traceable. Include a run ID in all log output:

```python
import uuid

@click.command()
def run_forecasts(config, run_target, scenario, issued_at):
    run_id = uuid.uuid4().hex[:8]
    logging.basicConfig(
        level=logging.INFO,
        format=f"%(asctime)s | {run_id} | %(levelname)-8s | %(name)s | %(message)s",
    )
    logger.info("Pipeline run started", extra={"run_id": run_id})
```

This makes it trivial to grep through logs for a specific run:

```bash
grep "a3f2c1b9" pipeline.log
```

## Duration Tracking

Track how long each phase takes:

```python
import time

class Timer:
    def __init__(self, label: str):
        self.label = label

    def __enter__(self):
        self._start = time.perf_counter()
        return self

    def __exit__(self, *exc):
        elapsed = time.perf_counter() - self._start
        logger.info("%s completed in %.2fs", self.label, elapsed)
        return False

# Usage
with Timer("Extract admin areas"):
    admin_areas = provider.fetch_admin_areas(country)

with Timer("Transform flood forecasts"):
    alerts = transform_floods(data)
```

## Structured Log Output (JSON)

For production pipelines that feed into log aggregation (Azure Monitor, ELK, etc.), use JSON output:

```python
import json
import logging

class JsonFormatter(logging.Formatter):
    def format(self, record):
        log_record = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "run_id": getattr(record, "run_id", None),
        }
        if record.exc_info:
            log_record["exception"] = self.formatException(record.exc_info)
        return json.dumps(log_record)

# Apply in production
handler = logging.StreamHandler()
handler.setFormatter(JsonFormatter())
logging.root.addHandler(handler)
```

### Use `extra` for structured fields

```python
logger.info(
    "Data source loaded",
    extra={
        "source": "glofas_discharge",
        "records": 1523,
        "country": "KEN",
        "duration_s": 3.42,
    },
)
```

## Azure Application Insights

For Azure-deployed pipelines, use `azure-monitor-opentelemetry`:

```python
import os
from azure.monitor.opentelemetry import configure_azure_monitor

def setup_azure_monitor() -> None:
    """Configure Azure Application Insights via OpenTelemetry."""
    conn_str = os.environ.get("APPLICATIONINSIGHTS_CONNECTION_STRING")
    if conn_str:
        configure_azure_monitor()
```

This sends logs, traces, and metrics to Application Insights without changing your logging calls.

> **Note:** The older `opencensus-ext-azure` package is deprecated. Use `azure-monitor-opentelemetry` for all new pipelines. See [../shared/logging.md](../shared/logging.md#azure-monitor-opentelemetry) for details.

## Alerting on Failure

Pipeline failures must be visible. At minimum:

1. **Non-zero exit code** — so schedulers (cron, Azure Functions, GitHub Actions) detect failure
2. **Error-level log** — so log aggregation can trigger alerts
3. **Optional: notification** — Slack/email/Teams webhook on failure

```python
import sys

def main():
    try:
        errors = orchestrator.run()
        if errors:
            logger.error("Pipeline completed with errors: %s", errors)
            notify_failure(errors)
            sys.exit(1)
        logger.info("Pipeline completed successfully")
    except Exception:
        logger.exception("Pipeline crashed")
        notify_failure(["Unhandled exception — see logs"])
        sys.exit(1)
```

## Log Levels for Pipelines

See [../shared/logging.md](../shared/logging.md#log-levels) for the universal log levels table. Pipeline-specific guidance:

| Level | Pipeline use | Example |
|-------|-------------|---------|
| `DEBUG` | Raw API responses, intermediate data shapes | `"API response: %s"` |
| `INFO` | Normal progress: loaded, transformed, submitted | `"Loaded 1523 records from GloFAS"` |
| `WARNING` | Recoverable: skipped record, fallback used | `"Station XYZ missing from thresholds, skipping"` |
| `ERROR` | Output affected: submission failed for one entity | `"Failed to submit data for KEN"` |
| `CRITICAL` | Pipeline cannot continue at all | `"Config file not found, aborting"` |

