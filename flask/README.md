# Flask Best Practices

Best practices for building Flask-based applications. Based on industry-standard Flask patterns and our own work.

## 🚀 Starting a new project

Copy [copilot-instructions.md](copilot-instructions.md) into your repo at `.github/copilot-instructions.md`. It consolidates the most important rules from this guide (and the [shared](../shared/) standards) into a single concise file that GitHub Copilot reads automatically.

## Guide Structure

| File | Topic |
|------|-------|
| [01-project-structure.md](01-project-structure.md) | Directory layout, blueprints, when to split |
| [02-app-configuration.md](02-app-configuration.md) | Application factory, extensions, middleware |
| [03-routes-and-templates.md](03-routes-and-templates.md) | Blueprints, Jinja2, forms, CSRF |
| [04-configuration-management.md](04-configuration-management.md) | Environment variables, settings classes, secrets |
| [05-error-handling.md](05-error-handling.md) | Error pages, exception handlers, flash messages |
| [06-security.md](06-security.md) | Headers, sessions, CSRF, input validation |
| [07-testing.md](07-testing.md) | Test client, fixtures, mocking external APIs |
| [08-docker.md](08-docker.md) | Flask Dockerfile, Gunicorn config (→ [shared/docker](../shared/docker.md)) |
| [09-ci-cd.md](09-ci-cd.md) | Zip deploy to Azure (→ [shared/ci-cd](../shared/ci-cd.md)) |
| [10-dependencies.md](10-dependencies.md) | Flask-specific deps (→ [shared/dependencies](../shared/dependencies.md)) |
| [11-logging-observability.md](11-logging-observability.md) | Flask logging hooks (→ [shared/logging](../shared/logging.md)) |
| [12-code-quality.md](12-code-quality.md) | Flask type patterns (→ [shared/code-quality](../shared/code-quality.md)) |

## Glossary

Flask-specific terms used throughout these guides. For shared terms (uv, Ruff, Gunicorn, Lockfile, etc.) see the [shared glossary](../shared/README.md#glossary).

| Term | Definition |
|------|-----------|
| **Application factory** | A function (conventionally `create_app()`) that constructs and configures a Flask app instance. Enables multiple configurations (dev, test, prod) from the same code and avoids global state. See [02-app-configuration.md](02-app-configuration.md). |
| **Blueprint** | A Flask object that groups related routes, templates, and static files into a reusable module. Blueprints make it easy to organize a growing app without circular imports. See [03-routes-and-templates.md](03-routes-and-templates.md). |
| **CSRF (Cross-Site Request Forgery)** | An attack where a malicious site tricks a user's browser into submitting a request to your app using the user's authenticated session. Prevented by embedding a per-session token in forms. See [06-security.md](06-security.md). |
| **CSP (Content Security Policy)** | An HTTP response header that tells the browser which sources of scripts, styles, images, etc. are allowed. Main defense against XSS. See [06-security.md](06-security.md). |
| **Deferred extension initialization** | The pattern of creating a Flask extension object (e.g., `CSRFProtect()`) at module level without binding it to an app, then calling `ext.init_app(app)` inside the factory. Avoids tight coupling. See [02-app-configuration.md](02-app-configuration.md). |
| **Flash message** | A one-time message stored in the session (via `flash()`) and displayed on the next page render. Used to give feedback after redirects ("Batch created successfully!"). See [05-error-handling.md](05-error-handling.md). |
| **Jinja2** | Flask's default templating engine. Renders HTML from `.html` template files with variables, loops, conditionals, inheritance, and macros. Auto-escapes output to prevent XSS. See [03-routes-and-templates.md](03-routes-and-templates.md). |
| **Middleware / `@after_request`** | Code that runs on every request or response. In Flask, this is typically implemented via `@app.before_request` and `@app.after_request` hooks rather than WSGI middleware. Used for security headers, logging, etc. See [02-app-configuration.md](02-app-configuration.md). |
| **PRG (Post/Redirect/Get)** | A pattern where a successful `POST` results in an HTTP redirect (302/303) to a `GET` page. Prevents duplicate form submissions if the user refreshes. See [03-routes-and-templates.md](03-routes-and-templates.md). |
| **WSGI (Web Server Gateway Interface)** | The Python standard ([PEP 3333](https://peps.python.org/pep-3333/)) defining how web servers communicate with Python apps. Flask is a WSGI application; Gunicorn is a WSGI server. See [08-docker.md](08-docker.md). |
| **WTForms / Flask-WTF** | A form-handling library that provides server-side validation, type coercion, CSRF token handling, and error messages for HTML forms. See [03-routes-and-templates.md](03-routes-and-templates.md). |
