# 05 — Error Handling

## Register Custom Error Handlers

Flask lets you register handlers for HTTP error codes and custom exceptions. Do this in the factory:

```python
# src/my_app/errors.py
from flask import render_template, flash, redirect, url_for, jsonify, request
import logging

logger = logging.getLogger(__name__)


def register_error_handlers(app):
    """Register all error handlers on the app."""

    @app.errorhandler(400)
    def bad_request(error):
        return _render_error(400, "Bad Request", str(error)), 400

    @app.errorhandler(403)
    def forbidden(error):
        return _render_error(403, "Forbidden", "You do not have permission to access this resource."), 403

    @app.errorhandler(404)
    def not_found(error):
        return _render_error(404, "Not Found", "The page you requested does not exist."), 404

    @app.errorhandler(500)
    def internal_error(error):
        logger.exception("Internal server error")
        return _render_error(500, "Internal Server Error", "Something went wrong. Please try again later."), 500

    @app.errorhandler(ExternalServiceError)
    def external_service_error(error):
        logger.error(f"External service error: {error}")
        return _render_error(502, "Service Unavailable", str(error)), 502


def _render_error(status_code: int, title: str, message: str):
    """Render HTML error page or JSON response based on Accept header."""
    if request.accept_mimetypes.best == "application/json":
        return jsonify({"error": {"code": status_code, "message": message}})
    return render_template("errors/error.html", code=status_code, title=title, message=message)
```

### Error template

```html
{# templates/errors/error.html #}
{% extends "base.html" %}

{% block title %}Error {{ code }}{% endblock %}

{% block content %}
<div class="error-page">
    <h1>{{ code }}</h1>
    <h2>{{ title }}</h2>
    <p>{{ message }}</p>
    <a href="{{ url_for('main.index') }}">
        <button type="button">Go back to Homepage</button>
    </a>
</div>
{% endblock %}
```

## Custom Exception Hierarchy

Define domain exceptions independent of Flask/HTTP:

```python
# src/my_app/exceptions.py

class AppError(Exception):
    """Base class for all application errors."""
    pass

class AuthenticationError(AppError):
    """Login credentials are invalid."""
    pass

class AuthorizationError(AppError):
    """User lacks required permissions."""
    pass

class ExternalServiceError(AppError):
    """An external API call failed."""
    pass

class ValidationError(AppError):
    """Input violates business rules (beyond form validation)."""
    pass
```

Then register handlers:
```python
@app.errorhandler(AuthenticationError)
def auth_error(error):
    flash(str(error), "error")
    return redirect(url_for("main.index"))

@app.errorhandler(AuthorizationError)
def authz_error(error):
    return _render_error(403, "Access Denied", str(error)), 403
```

## Flash Messages Over Error Pages

For user-facing errors that should return the user to a form, use **[flash messages](README.md#glossary)**:

❌ Bad — separate error page for every error:
```python
if not cookie:
    error = "Invalid credentials"
    return render_template("error.html", error=error)
```

✅ Good — flash and redirect back to the form:
```python
if not cookie:
    flash("Invalid credentials. Please check your username and password.", "error")
    return redirect(url_for("main.index"))
```

Display in the base template:
```html
{% with messages = get_flashed_messages(with_categories=true) %}
    {% if messages %}
        {% for category, message in messages %}
            <div class="alert alert-{{ category }}" role="alert">
                {{ message }}
            </div>
        {% endfor %}
    {% endif %}
{% endwith %}
```

### When to use what

| Scenario | Approach |
|----------|----------|
| Form validation failed | Re-render form with field errors (WTForms handles this) |
| Business logic error (bad credentials, wrong permissions) | `flash()` + redirect to form |
| Resource not found (404) | Error handler renders error template |
| Unexpected server error (500) | Error handler renders generic error page |
| External service down | Error handler for `ExternalServiceError`, log the details |

## Never Expose Internal Details

In production, error messages shown to users should be generic. Log the details server-side:

```python
@app.errorhandler(500)
def internal_error(error):
    # Log full traceback for debugging
    logger.exception("Internal server error")
    # Show generic message to user
    return _render_error(500, "Internal Server Error", "Something went wrong."), 500
```

❌ Bad — leaking implementation details:
```python
return render_template("error.html", error=f"Error: {data_request.status_code}\n{data_request.content}")
```

✅ Good — user-friendly message, details in logs:
```python
logger.error(f"121 API error: {response.status_code} {response.text}")
flash("Could not connect to 121. Please try again later.", "error")
```

## Raising Exceptions in Services

Service functions should raise domain exceptions. Route handlers (or error handlers) translate to user-facing messages:

```python
# services/api_121.py
def login_121(username: str, password: str, *, base_url: str) -> str:
    resp = requests.post(f"{base_url}/users/login", data={"username": username, "password": password})
    if resp.status_code == 401:
        raise AuthenticationError("Invalid 121 credentials")
    if not resp.ok:
        raise ExternalServiceError(f"121 login failed with status {resp.status_code}")
    return resp.json()["access_token_general"]
```
