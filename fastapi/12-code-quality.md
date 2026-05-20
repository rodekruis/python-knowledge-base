# 12 — Code Quality

> **Shared foundation:** See [../shared/code-quality.md](../shared/code-quality.md) for the canonical ruff config, type checking (ty/mypy), pre-commit setup, naming conventions, docstring style, anti-patterns, and Makefile template.

This addendum covers **FastAPI-specific** code quality patterns only.

---

## Type Hints in FastAPI

FastAPI leverages type hints for request validation, response serialization, and OpenAPI docs. They are **not optional**:

```python
# ✅ Good — typed, auto-documented, validated
async def search(payload: SearchRequest) -> SearchResponse:
    results: list[SearchResult] = await do_search(payload.query)
    return SearchResponse(results=results)

# ❌ Bad — no hints, no docs, no validation
async def search(payload):
    results = await do_search(payload.query)
    return {"results": results}
```

### FastAPI-Specific Type Patterns

```python
from fastapi import Depends, Query
from pydantic import BaseModel

class ItemCreate(BaseModel):
    name: str
    price: float

@router.post("/items", response_model=ItemResponse)
async def create_item(
    item: ItemCreate,
    db: AsyncSession = Depends(get_db),
    page: int = Query(default=1, ge=1),
) -> ItemResponse:
    ...
```

---

## Import Linter (Layer Enforcement)

Prevent architectural drift in FastAPI's layered structure:

```toml
# pyproject.toml
[tool.importlinter]
root_packages = ["my_app"]

[[tool.importlinter.contracts]]
name = "Domain has no project imports"
type = "independence"
modules = ["my_app.domain"]

[[tool.importlinter.contracts]]
name = "Layers"
type = "layers"
layers = [
    "my_app.api",
    "my_app.services",
    "my_app.domain",
]
```

This catches violations in CI — importing an adapter in the domain layer will fail the build.

```bash
uv add --dev import-linter
uv run lint-imports
```

---

## Ruff: Additional Rule for FastAPI

Consider adding `"T20"` (no `print()`) since FastAPI apps should use structured logging:

```toml
[tool.ruff.lint]
select = ["E", "W", "F", "I", "UP", "B", "SIM", "T20"]
```

See [../shared/code-quality.md](../shared/code-quality.md#ruff-linter--formatter) for the full canonical config.

