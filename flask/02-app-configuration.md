# 02 — App Configuration

## Application Factory

The core of a well-structured Flask app. Every Flask project should use an **[application factory](README.md#glossary)** (`create_app()`):

```python
# src/my_app/__init__.py
from flask import Flask
from .config import Config, TestConfig
from .extensions import csrf, login_manager

def create_app(config_class=Config):
    app = Flask(__name__)
    app.config.from_object(config_class)

    # Initialize extensions (deferred binding)
    csrf.init_app(app)

    # Register blueprints
    from .main.routes import main_bp
    app.register_blueprint(main_bp)

    # Register error handlers
    from .errors import register_error_handlers
    register_error_handlers(app)

    # Register after-request hooks
    from .middleware import register_after_request
    register_after_request(app)

    return app
```

### Why this matters

| Benefit | Explanation |
|---------|-------------|
| **Testability** | Create separate app instances with `TestConfig` — no shared state between tests |
| **Multiple configs** | Dev, staging, production share the same code, differ only in config class |
| **Circular imports** | Deferred blueprint registration avoids import cycles |
| **Extension flexibility** | Extensions bind to `app` at runtime, not at import time |

## Extensions Module

Use **[deferred extension initialization](README.md#glossary)** — declare extensions in a separate file, unbound. Bind them inside `create_app()`:

```python
# extensions.py
from flask_wtf.csrf import CSRFProtect

csrf = CSRFProtect()
# Add more: login_manager = LoginManager(), db = SQLAlchemy(), etc.
```

❌ Bad — binding at module level:
```python
app = Flask(__name__)
csrf = CSRFProtect(app)  # Tightly coupled, impossible to test with different config
```

✅ Good — deferred binding:
```python
csrf = CSRFProtect()     # Created once

def create_app():
    app = Flask(__name__)
    csrf.init_app(app)   # Bound to app in factory
    return app
```

## Security Headers via `@after_request`

Use **[middleware / `@after_request` hooks](README.md#glossary)** to formalize security headers as a reusable function:

```python
# src/my_app/middleware.py

def register_after_request(app):
    @app.after_request
    def add_security_headers(response):
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "0"  # Modern browsers: use CSP instead
        response.headers["Strict-Transport-Security"] = (
            "max-age=31536000; includeSubDomains"
        )
        response.headers["Content-Security-Policy"] = _build_csp()
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        response.headers["Permissions-Policy"] = (
            "camera=(), microphone=(), geolocation=()"
        )
        return response


def _build_csp():
    """Build a Content-Security-Policy header.

    Adjust directives based on what the app actually loads.
    """
    return "; ".join([
        "default-src 'self'",
        "script-src 'self'",
        "style-src 'self'",
        "img-src 'self' data:",
        "font-src 'self'",
        "frame-ancestors 'none'",
    ])
```

> **Note on `X-XSS-Protection`:** The `1; mode=block` value [can introduce vulnerabilities](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/X-XSS-Protection) in older browsers. Modern best practice is to set it to `0` and rely on `Content-Security-Policy` instead. Most modern browsers have deprecated this header entirely.

## Session Configuration

Configure sessions securely in the config class, not scattered through `app.py`:

```python
# config.py
class Config:
    SECRET_KEY = os.environ["SECRET_KEY"]         # Required, fail fast
    SESSION_COOKIE_SECURE = True                  # HTTPS only
    SESSION_COOKIE_HTTPONLY = True                 # No JS access
    SESSION_COOKIE_SAMESITE = "Lax"               # CSRF protection
    PERMANENT_SESSION_LIFETIME = 1800             # 30 minutes
```

> **Important:** `PERMANENT_SESSION_LIFETIME` only takes effect when `session.permanent = True` is set. Without this, the session cookie has no expiry (browser session only). Set it explicitly after login:
> ```python
> from flask import session
> session.permanent = True
> session["user_id"] = user.id
> ```

❌ Bad — conditional logic in global scope:
```python
if os.getenv("FLASK_ENV") == 'dev':
    app.config["SESSION_COOKIE_SECURE"] = False
else:
    app.config["SESSION_COOKIE_SECURE"] = True
```

✅ Good — separate config classes:
```python
class Config:
    SESSION_COOKIE_SECURE = True

class DevConfig(Config):
    SESSION_COOKIE_SECURE = False
    DEBUG = True

class TestConfig(Config):
    TESTING = True
    SESSION_COOKIE_SECURE = False
    WTF_CSRF_ENABLED = False  # Disable CSRF in tests
```

## WSGI Entry Point

Keep `wsgi.py` minimal — it just imports and exposes the app:

```python
# wsgi.py
from my_app import create_app

app = create_app()
```

This is what Gunicorn binds to: `gunicorn wsgi:app`.

## `.flaskenv` and `python-dotenv`

Flask supports `python-dotenv` natively. When installed, Flask auto-loads `.flaskenv` (Flask-specific defaults) and `.env` (secrets):

```dotenv
# .flaskenv — committed to git (no secrets!)
FLASK_APP=wsgi.py
FLASK_DEBUG=0
```

```dotenv
# .env — NOT committed (in .gitignore)
SECRET_KEY=super-secret-value
INSTANCE121=training
```

> **Important:** `.flaskenv` is for Flask CLI variables only (`FLASK_APP`, `FLASK_DEBUG`). Application secrets go in `.env`, which is git-ignored.
