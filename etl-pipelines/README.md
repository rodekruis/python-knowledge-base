# ETL Pipeline Best Practices

Best practices for building data pipelines (ETL/ELT). Based on industry-standard data engineering patterns and our own work.

## Scope

This guide covers **scheduled or triggered data pipelines** that:

1. **Extract** data from external sources (APIs, blob stores, databases, file servers)
2. **Transform** that data (filtering, aggregation, forecasting, model inference)
3. **Load** results into a target system (API, database, file system, blob storage)

It is target-agnostic: whether you push results to an alerting platform, a dashboard, a data warehouse, or a file share, the engineering principles are the same.

## 🚀 Starting a new project

Copy [copilot-instructions.md](copilot-instructions.md) into your repo at `.github/copilot-instructions.md`. It consolidates the most important rules from this guide (and the [shared](../shared/) standards) into a single concise file that GitHub Copilot reads automatically.

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

## Glossary

ETL and software terms used across this guide:

| Term | Definition |
|------|------------|
| **ETL** | ETL means Extract, Transform, Load: pull data from sources, process it, then publish it to a target system. In this guide, this is the core workflow for scheduled data pipelines. |
| **Orchestration** | Orchestration is the control layer that decides what runs, in what order, and with which configuration. It wires extract, transform, and load steps together and handles run-level errors and reporting. |
| **Protocol** | A Protocol in Python defines a behavior contract (what methods or attributes an object must have) without forcing class inheritance. It lets you swap components like extractors or loaders as long as they follow the same interface. |
| **ABC (Abstract Base Class)** | An Abstract Base Class defines required methods that child classes must implement. In this guide, storage backends use this so local and cloud storage can be used interchangeably. |
| **Concurrency (Async)** | Concurrency means running multiple tasks during the same time window, often with async I/O for network-heavy work. In ETL, this speeds up downloading or reading from many sources. |
| **Idempotency** | Idempotency means running the same pipeline multiple times with the same input gives the same result and does not create duplicates. This is critical for retries, backfills, and scheduler reliability. |
| **Dataclass** | A dataclass is a compact way to define structured, typed Python objects for data. The guide recommends converting raw API dictionaries into dataclasses early for safer, clearer code. |
| **Enum (StrEnum)** | An enum defines a fixed set of allowed values, such as run targets or output modes. This prevents typo-based bugs and makes config validation and autocomplete better. |
| **Integrity Checks** | Integrity checks are local validation rules run before writing output, such as coordinate ranges or required fields. They catch bad payloads early and avoid confusing downstream API errors. |
| **Run Target** | A run target is an environment profile such as debug, test, or prod that controls data sources and output behavior. It allows the same codebase to run safely in different environments by changing configuration, not code. |
