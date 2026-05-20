# 12 — Deployment and Scheduling

> **Shared foundation:** See [../shared/docker.md](../shared/docker.md) for universal Dockerfile principles (layer ordering, `--no-dev --frozen`, .dockerignore, health checks) and [../shared/ci-cd.md](../shared/ci-cd.md) for the standard lint-and-test workflow, action versions, and OIDC authentication.

This document covers **pipeline-specific** deployment: container entry points, scheduling options (cron, ACI, Azure Functions), environment separation via run targets, and CI/CD for the pipeline's own code.

---

## Docker for Pipeline Containers

### Dockerfile — Approach A: uv (recommended)

```dockerfile
FROM python:3.12-slim AS base

# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# Set working directory
WORKDIR /app

# Install dependencies (cached layer)
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-install-project

# Copy application code
COPY src/ src/
COPY configs/ configs/
RUN uv sync --frozen --no-dev

# Default entry point
ENTRYPOINT ["uv", "run", "run-pipeline"]
```

### Dockerfile — Approach B: poetry

```dockerfile
FROM python:3.13-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# Install poetry
RUN pip install poetry

# Install deps (cached layer)
COPY pyproject.toml poetry.lock ./
RUN poetry config virtualenvs.create false \
    && poetry install --no-interaction --no-ansi --no-root --without dev

# Copy application code
COPY . .
RUN poetry install --no-interaction --no-ansi --without dev

# Non-root user (good practice)
RUN useradd --create-home appuser
USER appuser

ENTRYPOINT ["python", "-m", "exposure_vulnerability_retrieval.main"]
```

**Both work.** uv is faster (10-100x install speed) and produces smaller images. Use poetry if your team already uses it and switching isn't worth the effort.

### Key principles

See [../shared/docker.md](../shared/docker.md) for the full rationale. Pipeline-specific notes:

| Principle | Pipeline implication |
|-----------|---------------------|
| `ENTRYPOINT` with CLI | Container *is* the pipeline — `docker run my-pipeline --config ... --run-target prod` |
| `--no-dev --frozen` | Reproducible builds; no test/lint deps in production |
| Install deps before copying code | Docker layer caching — deps don't change often |
| Include `configs/` | Pipeline YAML configs are part of the image |

### Run it

```bash
# Build
docker build -t my-pipeline .

# Run with config
docker run --env-file .env my-pipeline --config configs/floods.yaml --run-target prod

# Run a specific scenario (testing)
docker run my-pipeline --config configs/floods.yaml --run-target debug --scenario alert
```

## .dockerignore

See [../shared/docker.md](../shared/docker.md) for the canonical `.dockerignore`. For pipelines, also exclude:

```
output/
data/
configs/local/
```

## Scheduling Options

Pipelines need to run on a schedule. The right tool depends on your infrastructure.

### Option 1: GitHub Actions (simplest)

Best for: pipelines that run daily/hourly, don't need sub-minute precision, and are already in GitHub.

```yaml
# .github/workflows/run-pipeline.yml
name: Run Pipeline

on:
  schedule:
    - cron: "0 6 * * *"  # Daily at 06:00 UTC
  workflow_dispatch:       # Manual trigger

jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4

      - name: Run pipeline
        env:
          API_URL: ${{ secrets.API_URL }}
          API_TOKEN: ${{ secrets.API_TOKEN }}
        run: uv run run-pipeline --config configs/floods.yaml --run-target prod
```

**Advantages**: No infrastructure to manage, secrets built in, logs in GitHub.
**Disadvantages**: 6-hour max runtime, no sub-minute scheduling, cold start overhead.

### Option 2: Azure Container Instances (ACI)

Best for: pipelines that need more control, longer runtimes, or Azure integration.

```bash
az container create \
  --resource-group my-rg \
  --name flood-pipeline \
  --image myregistry.azurecr.io/my-pipeline:latest \
  --restart-policy Never \
  --environment-variables API_URL=$API_URL \
  --secure-environment-variables API_TOKEN=$API_TOKEN \
  --command-line "uv run run-pipeline --config configs/floods.yaml --run-target prod"
```

Trigger on schedule with Azure Logic Apps or a simple cron-based Azure Function.

### Option 2b: Azure Logic Apps

Azure Logic Apps can trigger ACI on a schedule. A Logic App definition (`logic_app/`) defines the recurrence trigger and the ACI create action. This is a good serverless alternative to Azure Functions for orchestrating container runs — it requires zero code, just an ARM template.

### Option 3: Azure Functions (Timer Trigger)

Best for: lightweight pipelines or pipeline orchestrators that trigger other containers.

```python
# function_app.py
import azure.functions as func
import subprocess

app = func.FunctionApp()

@app.timer_trigger(schedule="0 0 6 * * *", arg_name="timer")
def run_pipeline(timer: func.TimerRequest):
    """Trigger pipeline daily at 06:00 UTC."""
    result = subprocess.run(
        ["uv", "run", "run-pipeline", "--config", "configs/floods.yaml", "--run-target", "prod"],
        capture_output=True, text=True,
    )
    if result.returncode != 0:
        raise RuntimeError(f"Pipeline failed:\n{result.stderr}")
```

### Option 4: Cron on a VM

Simplest, but you own the infrastructure:

```cron
# /etc/cron.d/pipeline
0 6 * * * appuser cd /opt/pipeline && docker compose run --rm pipeline --config configs/floods.yaml --run-target prod >> /var/log/pipeline.log 2>&1
```

### Comparison

| Feature | GitHub Actions | ACI | Azure Functions | Cron on VM |
|---------|---------------|-----|----------------|------------|
| Setup effort | ⭐ low | ⭐⭐ medium | ⭐⭐ medium | ⭐⭐⭐ high |
| Max runtime | 6 hours | Unlimited | 10 min (consumption) | Unlimited |
| Cost | Free (public repos) | Pay per run | Pay per execution | Fixed VM cost |
| Secrets | GitHub Secrets | Env vars | Key Vault | Env vars |
| Monitoring | GitHub UI | Azure Monitor | Azure Monitor | DIY |
| Retry on failure | ✅ (rerun) | Manual | ✅ (built-in) | Manual |

## CI/CD for the Pipeline Itself

The pipeline's own code needs CI/CD too.

### `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint-and-type-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync --frozen
      - run: uv run ruff check .
      - run: uv run ruff format --check .
      - run: uv run pyright

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync --frozen
      - run: uv run pytest tests/unit/ -v
      - run: uv run pytest tests/integration_infra/ -v --timeout=120

  build:
    runs-on: ubuntu-latest
    needs: [lint-and-type-check, test]
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: myregistry.azurecr.io
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}
      - uses: docker/build-push-action@v6
        with:
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: myregistry.azurecr.io/my-pipeline:latest
```

## Environment Separation

Use run targets to separate environments cleanly:

| Run Target | Config | Output Mode | API Endpoint | Scheduling |
|-----------|--------|------------|-------------|------------|
| `debug` | `configs/floods.yaml` | `local` (files) | N/A | Manual |
| `test` | `configs/floods.yaml` | `api` | Staging API | On PR merge |
| `prod` | `configs/floods.yaml` | `api` | Production API | Daily cron |

```bash
# Development
uv run run-pipeline --config configs/floods.yaml --run-target debug

# Staging (in CI after merge)
uv run run-pipeline --config configs/floods.yaml --run-target test

# Production (scheduled)
uv run run-pipeline --config configs/floods.yaml --run-target prod
```

