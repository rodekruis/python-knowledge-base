# 08 — Error Handling

## Error Strategy: Accumulate, Don't Abort

Pipelines that process multiple entities should **collect errors and continue**, not fail on the first problem. One entity's failure shouldn't block the others:

```python
def run_pipeline(config_path, run_target) -> list[str]:
    all_errors: list[str] = []

    for entity in entities:
        try:
            errors = _run_entity(entity, ...)
            all_errors.extend(errors)
        except Exception as exc:
            logger.exception(f"Unexpected error for {entity.id}")
            all_errors.append(f"{entity.id}: {exc}")

    return all_errors  # Empty list = success
```

### When to fail fast vs accumulate

| Fail fast (abort immediately) | Accumulate (continue, report at end) |
|------------------------------|--------------------------------------|
| Config file is invalid | One of 5 countries fails to load data |
| Required environment variable is missing | One data source returns empty results |
| Transform function registration fails | One entity's output fails integrity checks |
| Database connection cannot be established | API submission fails for one entity |

## Error Return Pattern

Approach A uses a clean pattern: functions return `list[str]` where an empty list means success:

```python
def _run_entity(entity, ...) -> list[str]:
    """Process one entity. Returns list of error messages."""
    # Extract
    if not provider.load_data(entity):
        return [f"Failed to load data for {entity.id}"]

    # Transform
    submitter = DataSubmitter(api_client)
    transform_fn(provider, submitter, entity.id, entity.level)

    # Load
    return submitter.send_all(entity.output_mode, entity.output_path)
```

This is simpler than exceptions for expected pipeline failures (missing data, validation errors). Reserve exceptions for truly unexpected situations (bugs, infra failures).

## Structured Error Messages

Include context in every error message:

❌ Bad:
```python
return ["Invalid data"]
return ["API error"]
return ["Something went wrong"]
```

✅ Good:
```python
return [f"Entity '{entity_id}': no station data available for source '{source}'"]
return [f"Entity '{entity_id}': API returned {status_code}: {response_text[:200]}"]
return [f"Entity '{entity_id}': centroid latitude {lat} is outside [-90, 90]"]
```

Pattern: **`{entity}: {what happened} {relevant values}`**

## DataSubmitter Error Accumulation

The submitter should collect errors from both domain code and integrity checks:

```python
class DataSubmitter:
    def __init__(self):
        self.errors: dict[str, str] = {}

    def add_error(self, error: str) -> None:
        """Called by domain code when transform encounters a problem."""
        self.errors[f"domain:{len(self.errors)}"] = error

    def send_all(self, output_mode, output_path) -> list[str]:
        # Check domain errors first
        if self.errors:
            return list(self.errors.values())

        # Then run integrity checks
        integrity_errors = self._check_integrity()
        if integrity_errors:
            return integrity_errors

        # All clean — send
        return self._dispatch_output(output_mode, output_path)
```

## Exit Codes

The CLI entry point should translate errors to exit codes:

```python
@click.command()
def main(config_path, run_target, ...):
    errors = run_pipeline(config_path, run_target)

    if errors:
        logger.error(f"Pipeline completed with {len(errors)} error(s):")
        for error in errors:
            logger.error(f"  - {error}")
        sys.exit(1)    # Non-zero exit code signals failure

    logger.info("Pipeline completed successfully")
    sys.exit(0)
```

Exit codes matter because:
- **Cron/schedulers** check exit codes to determine success/failure
- **CI/CD** marks the step as failed
- **Monitoring** alerts on non-zero exit codes
- **Retry logic** only retries on failure

## Never Swallow Exceptions Silently

❌ Bad:
```python
try:
    response = requests.get(url)
    data = response.json()
except Exception:
    pass  # Silently ignores all errors
```

✅ Good:
```python
try:
    response = session.get(url, timeout=30)
    response.raise_for_status()
    data = response.json()
except requests.RequestException as exc:
    logger.error(f"Failed to fetch {url}: {exc}")
    container.error = str(exc)
    return
```

## Sensitive Data in Error Messages

Don't include credentials, tokens, or PII in error messages:

❌ Bad:
```python
logger.error(f"Login failed for user {username} with password {password}")
logger.error(f"API call failed with headers: {headers}")  # Includes auth token
```

✅ Good:
```python
logger.error(f"Login failed for user '{username}'")
logger.error(f"API call to {url} failed with status {status_code}")
```