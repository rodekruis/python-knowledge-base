# Dependencies

## Package Manager: uv

Use **[uv](https://docs.astral.sh/uv/)** for all dependency management. It replaces pip, pip-tools, and virtualenv with a single, fast tool.

### Essential Commands

```bash
# Create / sync virtual environment
uv sync              # Install all deps (including dev)
uv sync --no-dev     # Production install (no test/lint tools)

# Add / remove packages
uv add httpx         # Add a runtime dependency
uv add --dev pytest  # Add a dev-only dependency
uv remove httpx      # Remove a package

# Run without activating
uv run pytest        # Run a command in the venv
uv run python main.py

# Update lockfile
uv lock              # Re-resolve and write uv.lock
uv lock --upgrade    # Upgrade all packages to latest compatible
```

### Why uv?

| Feature | uv | pip + pip-tools |
|---------|-----|-----------------|
| Speed | 10-100× faster | Baseline |
| Lockfile | `uv.lock` (automatic) | Manual `pip-compile` |
| Resolver | Deterministic, cross-platform | Platform-specific |
| Python management | `uv python install 3.12` | External (pyenv) |

---

## pyproject.toml Template

One canonical template for all Python projects:

```toml
[project]
name = "my-app"
version = "0.1.0"
description = "Short description of the project."
requires-python = ">=3.12"
dependencies = [
    # Runtime dependencies here
]

[dependency-groups]
dev = [
    "pytest>=8.0",
    "pytest-cov>=6.0",
    "ruff>=0.8",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/my_app"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--strict-markers --tb=short -q"

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes
    "I",    # isort
    "UP",   # pyupgrade
    "B",    # bugbear
    "SIM",  # simplify
    "T20",  # print statements
    "N",    # pep8-naming
    "RET",  # return statements
    "PTH",  # pathlib
]

[tool.ruff.lint.isort]
known-first-party = ["my_app"]
```

### Design Decisions

| Choice | Rationale |
|--------|-----------|
| `[dependency-groups]` | uv-native; cleaner than `[project.optional-dependencies]` for dev tools |
| `requires-python = ">=3.12"` | Latest stable; enables modern syntax (`match`, `type` aliases) |
| `line-length = 100` | Balanced readability on modern screens and side-by-side diffs |
| `hatchling` build backend | Lightweight, well-maintained, supports `src/` layout |

---

## Lockfile: `uv.lock`

- **Always commit `uv.lock`** to version control
- CI/Docker use `uv sync --frozen` — fails if lockfile is stale
- Developers run `uv lock` after editing `pyproject.toml`

### OneDrive Compatibility

If your workspace is on OneDrive, the `.venv/` folder causes sync issues. Add to `.gitignore` and tell VS Code to look in the right place:

```json
// .vscode/settings.json
{
    "python.defaultInterpreterPath": "${workspaceFolder}/.venv/Scripts/python.exe"
}
```

Alternatively, configure uv to place venvs outside OneDrive:

```bash
export UV_CACHE_DIR="C:/Users/<you>/.uv-cache"
```

---

## Version Pinning Strategy

| Dependency type | Pinning | Example |
|----------------|---------|---------|
| Direct runtime | Minimum with compatible range | `httpx>=0.27"` |
| Direct dev | Minimum with compatible range | `pytest>=8.0"` |
| Transitive (lockfile) | Exact (automatic via `uv.lock`) | `anyio==4.6.2` |

**Never pin exact versions in `pyproject.toml`** — that's what the lockfile is for.
The `pyproject.toml` specifies _intent_ (compatible range); `uv.lock` pins _reality_ (exact resolved versions).

---

## Dependabot / Renovate

Automate dependency updates:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "pip"   # works with pyproject.toml
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      minor-and-patch:
        update-types: ["minor", "patch"]
```

Review grouped PRs weekly. Pin major version bumps manually after testing.

---

## Migration from requirements.txt

```bash
# 1. Generate pyproject.toml from existing requirements
uv init --name my-app

# 2. Add each dependency (uv will resolve versions)
uv add $(cat requirements.txt | grep -v '^#' | grep -v '^$')

# 3. Verify everything resolves
uv lock

# 4. Remove old files
rm requirements.txt requirements-dev.txt

# 5. Update CI and Dockerfile to use uv
```

After migration, update your Dockerfile to use `uv sync --no-dev --frozen` instead of `pip install -r requirements.txt`.
