# 10 — CLI and Entrypoints

## Use Click for Pipeline CLIs

Every pipeline should expose a CLI. [Click](https://click.palletsprojects.com/) is the standard — it's explicit, composable, and supports type validation on arguments.

### ❌ Bad: argparse spaghetti

```python
import argparse
import sys

parser = argparse.ArgumentParser()
parser.add_argument("--config", required=True)
parser.add_argument("--environment", required=True)
parser.add_argument("--scenario", required=False)
parser.add_argument("--issued-at", required=False)
args = parser.parse_args()

if args.environment not in ["debug", "test", "prod"]:
    print("Invalid environment")
    sys.exit(1)
```

### ✅ Good: Click with type-safe options

```python
import click
from my_pipeline.infra.config_reader import Environment

@click.command()
@click.option(
    "--config",
    required=True,
    type=click.Path(exists=True, path_type=Path),
    help="Path to YAML configuration file.",
)
@click.option(
    "--environment",
    required=True,
    type=click.Choice([e.value for e in Environment], case_sensitive=False),
    help="Which environment to execute.",
)
@click.option(
    "--scenario",
    required=False,
    type=click.Choice(["alert", "no-alert"]),
    help="Use scenario data instead of live domain logic.",
)
@click.option(
    "--issued-at",
    required=False,
    type=click.DateTime(formats=["%Y-%m-%d"]),
    help="Override the pipeline run date (for backfills).",
)
def run_forecasts(config: Path, environment: str, scenario: str | None, issued_at: datetime | None):
    """Run the ETL pipeline for the given config and environment."""
    env = Environment(environment)
    orchestrator = Orchestrator(config, env, scenario=scenario, issued_at=issued_at)
    orchestrator.run()

if __name__ == "__main__":
    run_forecasts()
```

## Key CLI Options

Every pipeline CLI should support at least:

| Option | Purpose | Example |
|--------|---------|---------|
| `--config` | Pipeline config file | `configs/floods.yaml` |
| `--environment` | Environment tier | `debug`, `test`, `prod` |
| `--scenario` | Test scenario (bypass domain logic) | `alert`, `no-alert` |
| `--issued-at` | Override run date | `2024-01-15` |
| `--dry-run` | Validate config + extract, skip load | _(flag)_ |

### `--dry-run` is underused

```python
@click.option("--dry-run", is_flag=True, help="Validate and extract, but don't load.")
def run_forecasts(config, environment, scenario, issued_at, dry_run):
    orchestrator = Orchestrator(config, env, scenario=scenario, issued_at=issued_at)
    orchestrator.run(dry_run=dry_run)
```

## Register as a pyproject.toml Script

Always expose the CLI as an installable entry point:

```toml
# pyproject.toml
[project.scripts]
run-pipeline = "my_pipeline.infra.run_forecasts:run_forecasts"
```

After `uv sync`, users can run:

```bash
uv run run-pipeline --config configs/floods.yaml --environment debug
```

## Debug Entrypoints in Modules

For quick development iteration, add `if __name__ == "__main__"` blocks that exercise a single component:

```python
# my_pipeline/infra/config_reader.py

class ConfigReader:
    ...

if __name__ == "__main__":
    reader = ConfigReader()
    reader.read(Path("configs/floods.yaml"))
    print(reader.summary())
```

```python
# my_pipeline/infra/data_provider.py

class DataProvider:
    ...

if __name__ == "__main__":
    # Quick smoke test: read config, extract one data source
    reader = ConfigReader()
    reader.read(Path("configs/floods.yaml"))
    provider = DataProvider(client=ApiClient.from_env())
    provider.get_data(reader.countries[0].data_sources[:1])
    print("Loaded:", list(provider.loaded_data.keys()))
```

These are **not** the production entry point — they're for developer convenience. They should:
- Never be imported by production code
- Never contain important logic
- Be skippable by test runners

## Alternative: tyro for Dataclass-Driven CLIs

[tyro](https://brentyi.github.io/tyro/) generates CLI arguments directly from a `@dataclass`. If your config is already a dataclass (see [03 — Configuration](03-configuration.md)), this eliminates the duplication between config schema and CLI definition.

### ✅ Good: tyro with RunConfig dataclass

```python
from dataclasses import dataclass

@dataclass
class RunConfig:
    """Configure an ETL pipeline run.

    Attributes:
        country: ISO3 country code to process.
        extract: If True, run extraction step.
        transform: If True, run transformation step.
        load: If True, run load step to API.
    """
    country: str
    extract: bool = True
    transform: bool = True
    load: bool = True

# main.py
import tyro

def main():
    config = tyro.cli(RunConfig)
    # config.country, config.extract, etc. are typed and validated
    pipeline = ETLPipeline.from_config(config)
    pipeline.run()

if __name__ == "__main__":
    main()
```

This gives you:

```bash
python main.py --country SOM --no-extract --no-load
python main.py --help  # auto-generated docs from docstring + type hints
```

Booleans automatically become `--extract/--no-extract` flag pairs.

### When to use tyro vs Click

| Factor | Click | tyro |
|--------|-------|------|
| Config already in YAML | ✅ Click reads path to file | Overkill — just use `--config path` |
| Config is a dataclass | Duplicates fields in decorators | ✅ Zero duplication |
| Needs sub-commands (multi-pipeline) | ✅ `click.group()` is excellent | Possible but less ergonomic |
| Team familiarity | Most know Click | Less known, but simpler |
| Complex validation | ✅ Callbacks, custom types | Limited — validate after parse |

**Our recommendation**: Use **Click** when the pipeline reads a config file (Approach A pattern). Use **tyro** when the CLI *is* the config (Approach B pattern, small pipelines) — it's less code and harder to get wrong.

## Click Groups for Multi-Pipeline Repos

If a repo contains multiple pipelines (floods, drought, etc.), use Click groups:

```python
import click

@click.group()
def cli():
    """ETL pipeline manager."""
    pass

@cli.command()
@click.option("--config", required=True, type=click.Path(exists=True, path_type=Path))
@click.option("--environment", required=True)
def floods(config, environment):
    """Run the flood forecasting pipeline."""
    ...

@cli.command()
@click.option("--config", required=True, type=click.Path(exists=True, path_type=Path))
@click.option("--environment", required=True)
def drought(config, environment):
    """Run the drought forecasting pipeline."""
    ...
```

```toml
[project.scripts]
pipeline = "my_pipeline.cli:cli"
```

```bash
uv run pipeline floods --config configs/floods.yaml --environment debug
uv run pipeline drought --config configs/drought.yaml --environment prod
```

## CLI Exit Codes

Use consistent exit codes so schedulers and CI can react:

```python
import sys

EXIT_SUCCESS = 0
EXIT_PIPELINE_ERROR = 1  # Pipeline ran but produced errors
EXIT_CONFIG_ERROR = 2    # Bad config, never started

@click.command()
def run_forecasts(config, environment, scenario, issued_at):
    try:
        orchestrator = Orchestrator(config, env, scenario=scenario, issued_at=issued_at)
        errors = orchestrator.run()
        if errors:
            logger.error("Pipeline completed with %d errors", len(errors))
            sys.exit(EXIT_PIPELINE_ERROR)
        sys.exit(EXIT_SUCCESS)
    except ConfigError as exc:
        logger.error("Configuration error: %s", exc)
        sys.exit(EXIT_CONFIG_ERROR)
```
