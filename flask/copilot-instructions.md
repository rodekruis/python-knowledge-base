# Copilot Instructions — Flask Web App

Build server-rendered Flask web apps (Jinja2 templates, forms, sessions) for NLRC 510. Follow these
conventions exactly. They are the consolidated 510 best practices; deviate only with a documented reason.

## Working with the Engineer (Responsible AI Use)

Follow these principles: you assist, the human decides, reviews, and owns the output.

- **Think with the engineer, not for them.** Before generating non-trivial code, briefly discuss the
  approach and trade-offs. Surface design decisions rather than silently picking one.
- **Ask clarifying questions.** If a request is ambiguous, under-specified, or looks like the wrong
  approach, ask first — do not guess and produce a large answer. A short question beats a long wrong
  implementation.
- **Protect the engineer's understanding.** Don't just hand over answers to questions the engineer can
  reason through. Prompt them to attempt it, explain *why* a solution works, and call out edge cases.
  Optimize for long-term comprehension over speed.
- **Don't reinvent the wheel.** Prefer well-maintained existing libraries over bespoke implementations.
  Before writing custom code for a common problem (parsing, retries, validation, HTTP, dates), check
  whether the stdlib or an already-used dependency solves it, and say why a library is better.
- **Small, reviewable changes.** Keep diffs focused; separate refactors from features. Never produce
  massive, opaque changes.
- **Be transparent.** AI-assisted contributions should be disclosed (e.g. commit trailers) and must be
  fully understood by the human who commits them.
- **No secrets or PII into prompts/tools.** Never paste beneficiary, donor, or personnel data into AI
  tooling. Treat all generated code as untrusted until reviewed.

## Stack & Tooling

- **Python 3.12+**, managed with **uv** (never pip/poetry/virtualenv).
- Lint/format with **ruff**; type-check with **ty** (fallback **mypy**).
- Test with **pytest** + Flask `test_client`.
- Run production with **gunicorn** (WSGI); deploy via **zip deploy** to Azure App Service (or Docker).
- License: **Apache-2.0**. Built by [NLRC 510](https://www.510.global/).

```bash
uv sync                 # install (incl. dev)
uv add flask            # add runtime dep
uv add --dev pytest     # add dev dep
uv run pytest           # test
uv run ruff check .     # lint
uv run ty check         # type check
```

## Project Structure

Use a `src/` layout with an **application factory** and **blueprints**. Services contain logic; routes are
thin wiring. Services and clients **never import from `flask`**.

```
src/my_app/
├── __init__.py          # create_app() factory
├── config.py            # Config / DevConfig / TestConfig / ProdConfig
├── extensions.py        # unbound extensions (csrf = CSRFProtect())
├── middleware.py        # @after_request hooks (security headers, request logging)
├── errors.py            # register_error_handlers(app)
├── main/                # blueprint: routes.py + forms.py (WTForms)
├── auth/                # blueprint: routes.py
├── services/            # business logic, no Flask imports → raise domain exceptions
├── clients/             # external API wrappers (pure fns, injectable base_url)
├── templates/{base.html, main/, errors/, macros/}
└── static/{css,js,img}/
wsgi.py                  # Gunicorn entry: from my_app import create_app; app = create_app()
tests/{conftest.py, test_routes.py, test_services.py, test_forms.py, test_clients.py}
```

Flat `app.py` is acceptable only for single-purpose tools with ≤3 routes / <150 lines; graduate to
blueprints immediately past that.

## Application Factory

- Always use `create_app(config_class=Config)` even for small apps. Bind extensions with `init_app(app)`
  (define them unbound in `extensions.py` — never `CSRFProtect(app)` at module level).
- Register blueprints, error handlers, `@after_request` hooks, and logging inside the factory.
- `wsgi.py` is what Gunicorn binds to and must stay minimal.

```python
def create_app(config_class=Config):
    app = Flask(__name__)
    app.config.from_object(config_class)
    configure_logging(app)
    csrf.init_app(app)
    from .main.routes import main_bp
    app.register_blueprint(main_bp)
    register_error_handlers(app)
    register_after_request(app)
    return app
```

## Routes, Forms & Templates

- A handler does exactly three things: validate input → call a service → return a response. No business
  logic in routes.
- **Validate all input with WTForms** (`FlaskForm`) for type coercion, server-side validators, and CSRF.
- Follow **PRG** (Post/Redirect/Get): redirect after a successful POST.
- Use Jinja2 template inheritance from `base.html`; auto-escaping is on — only use `| safe` for
  self-generated trusted HTML. Render form fields via shared macros. Use `url_for` for links/static assets.
- Use **`flash()` + redirect** for user-facing business errors; reserve error templates for 404/500/etc.

## Configuration

- Config **classes** in `config.py` with inheritance (`Config` → `Dev/Test/Prod`). Required secrets via
  `os.environ["SECRET_KEY"]` (no default → fail fast). `TestConfig` sets `WTF_CSRF_ENABLED = False`.
- Access config via `current_app.config[...]`, never `os.getenv` deep in code. Keep services config-injectable.
- File tracking: `.flaskenv` (committed, Flask CLI vars only) · `.env` (gitignored secrets) ·
  `example.env` (committed template with comments).

## Security

- Set security headers via `@after_request`: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`,
  HSTS, `Content-Security-Policy` (tune per app), `Referrer-Policy`, `Permissions-Policy`.
  **Drop `X-XSS-Protection`** (deprecated) — rely on CSP.
- Session cookies: `SECURE=True`, `HTTPONLY=True`, `SAMESITE="Lax"`, finite `PERMANENT_SESSION_LIFETIME`
  (set `session.permanent = True` after login). Store only IDs/flags in the session.
- **CSRF on all forms** (`form.hidden_tag()` or `csrf_token()`; `X-CSRFToken` header for AJAX/htmx).
- Generate `SECRET_KEY` with `secrets.token_hex(32)`. Parameterize all SQL. Validate path converters
  (`<int:id>`). Never put credentials in URLs. Use `TRUSTED_HOSTS` (Flask 3.1+); add rate limiting where relevant.

## Error Handling

- Register handlers (`@app.errorhandler`) for 400/403/404/500 and custom exceptions via
  `register_error_handlers(app)`. Render HTML or JSON based on `Accept`. Log 500s with `logger.exception`.
- Define an `AppError` hierarchy (`AuthenticationError`, `AuthorizationError`, `ExternalServiceError`,
  `ValidationError`). Services raise these; routes translate to flash messages / responses.
- **Never expose internal details** to users — log them, show friendly messages.

## Testing

- Fixtures in `conftest.py`: `app` (built with `TestConfig`), `client`, `runner`.
- Test routes (GET/POST with form data, `follow_redirects=True`), services (pure, no Flask), forms
  (`app.test_request_context()`), and clients (mock HTTP with `responses`). Use `client.session_transaction()`
  to seed session data. Use `pytest-mock` (`mocker`) to patch services.
- Targets: 90%+ on services, 80%+ overall, 100% on auth/permissions.

```bash
uv run pytest --cov=my_app --cov-report=term-missing
uv run pytest -m "not integration"
```

## Logging & Observability

- One logger per module. Configure in `create_app` (`configure_logging(app)`): DEBUG if `app.debug` else
  INFO; silence `werkzeug`, `urllib3`. Log structured fields via `extra=`. **Never log secrets/PII.**
- Request logging via `@before_request`/`@after_request` (method, path, status, duration_ms); skip `/health`.
- Production telemetry: prefer **`azure-monitor-opentelemetry`** for new projects; the legacy
  `opencensus-ext-flask`/`opencensus-ext-azure` packages are **deprecated** — migrate when possible.
- Expose `GET /health` → `{"status": "ok"}` (untracked in logs).

## Docker & Deployment

WSGI app served by **gunicorn** (workers ≈ `2*cores+1`). Bind Azure's injected `PORT`. Never run
`app.run()` in production.

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
COPY pyproject.toml uv.lock ./
RUN uv sync --no-editable --no-dev --frozen
COPY src/ src/
COPY wsgi.py ./
EXPOSE 8000
CMD [".venv/bin/gunicorn", "wsgi:app", "--bind", "0.0.0.0:8000", "--workers", "2", "--timeout", "120"]
```

## CI/CD (GitHub Actions)

- `lint-and-test` on push/PR: `uv sync` → `ruff check` → `ruff format --check` → `ty check` →
  `pytest --cov`. Pin actions to major versions.
- Deploy: **zip deploy** to Azure App Service (`uv sync --no-dev --frozen` → zip excluding
  `.git tests .github *.md .env .flaskenv .venv` → `azure/webapps-deploy`). Use Docker for complex apps.
  Authenticate with **Azure OIDC**. Protect `main`; use GitHub Environments.

## pyproject.toml essentials

```toml
[project]
requires-python = ">=3.12"
dependencies = ["flask>=3.1,<4", "flask-wtf>=1.2,<2", "gunicorn>=22,<23", "python-dotenv>=1,<2"]

[dependency-groups]
dev = ["pytest>=8.0", "pytest-cov>=6.0", "pytest-mock", "responses", "ruff>=0.8", "ty"]

[tool.ruff]
line-length = 100
target-version = "py312"
[tool.ruff.lint]
select = ["E","W","F","I","UP","B","SIM","T20","N","RET","PTH"]
[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101"]
```

**Always commit `uv.lock`**. On OneDrive set `UV_LINK_MODE=copy`.

## Anti-Patterns

- `CSRFProtect(app)` at module level
- Business logic in routes
- `flask` imports in services
- `os.getenv` deep in code
- Rendering instead of redirecting after POST
- `| safe` on user input
- Exposing internal errors
- `print()` (use logging)
- Logging secrets
- `app.run()` in production
- Custom session/CSRF/auth handling where `Flask-WTF`/`WTForms`/`Flask-Login` already solve it
