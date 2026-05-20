# ETL Pipeline Best Practices

Best practices for building data pipelines (ETL/ELT). Based on industry-standard data engineering patterns and our own work.

## Scope

This guide covers **scheduled or triggered data pipelines** that:

1. **Extract** data from external sources (APIs, blob stores, databases, file servers)
2. **Transform** that data (filtering, aggregation, forecasting, model inference)
3. **Load** results into a target system (API, database, file system, blob storage)

It is target-agnostic: whether you push results to an alerting platform, a dashboard, a data warehouse, or a file share, the engineering principles are the same.

## Guide Structure

| File | Topic |
|------|-------|
| [01-project-structure.md](01-project-structure.md) | Directory layout, infra/domain separation, when to split |
| [02-pipeline-architecture.md](02-pipeline-architecture.md) | Orchestration, Protocol-based ETL, Storage abstraction, async execution |
| [03-configuration.md](03-configuration.md) | YAML vs Python config, run targets, country config Protocol, TypedDict |
| [04-data-types-and-validation.md](04-data-types-and-validation.md) | Typed dataclasses, enums, data integrity checks |
| [05-extract.md](05-extract.md) | Fetching from APIs, Storage abstraction, async downloads, Pydantic validation |
| [06-transform.md](06-transform.md) | Business logic separation, idempotency, pandas vs pure Python |
| [07-load.md](07-load.md) | Output modes, API submission, file output, integrity checks before send |
| [08-error-handling.md](08-error-handling.md) | Fail-fast vs accumulate, error reporting, exit codes |
| [09-testing.md](09-testing.md) | Unit tests, integration tests, scenario-based testing, fixtures |
| [10-cli-and-entrypoints.md](10-cli-and-entrypoints.md) | Click vs tyro CLI, script entry points, `pyproject.toml` scripts |
| [11-logging-observability.md](11-logging-observability.md) | loguru vs stdlib, run traceability, alerting (→ [shared/logging](../shared/logging.md)) |
| [12-deployment-and-scheduling.md](12-deployment-and-scheduling.md) | Docker, cron, GitHub Actions, Azure Functions, CI/CD (→ [shared/docker](../shared/docker.md)) |
| [13-dependencies-and-code-quality.md](13-dependencies-and-code-quality.md) | deptry, vulture, pyright, quality runner (→ [shared/dependencies](../shared/dependencies.md), [shared/code-quality](../shared/code-quality.md)) |
