# 04 — Configuration Management

## Use Config Classes

Flask has built-in support for config classes. Use them instead of scattering `os.getenv()` calls:

```python
# config.py
import os

class Config:
    """Base configuration — shared across all environments."""

    # Required — fail fast if missing
    SECRET_KEY = os.environ["SECRET_KEY"]

    # Session security
    SESSION_COOKIE_SECURE = True
    SESSION_COOKIE_HTTPONLY = True
    SESSION_COOKIE_SAMESITE = "Lax"
    PERMANENT_SESSION_LIFETIME = 1800  # 30 minutes

    # App-specific
    INSTANCE_121 = os.environ.get("INSTANCE_121", "")


class DevConfig(Config):
    """Local development overrides."""
    DEBUG = True
    SESSION_COOKIE_SECURE = False  # No HTTPS locally


class TestConfig(Config):
    """Test environment — deterministic, no external services."""
    TESTING = True
    DEBUG = True
    SESSION_COOKIE_SECURE = False
    WTF_CSRF_ENABLED = False          # Disable CSRF in tests
    SECRET_KEY = "test-secret-key"    # Fixed, deterministic


class ProdConfig(Config):
    """Production — strictest settings."""
    SESSION_COOKIE_SECURE = True
    # All secrets come from env vars or Azure Key Vault
```

Load in the factory:
```python
def create_app(config_class=Config):
    app = Flask(__name__)
    app.config.from_object(config_class)
    return app
```

### Benefits

- **Fail fast**: `os.environ["SECRET_KEY"]` (without default) raises `KeyError` at startup, not on first request
- **Testable**: `create_app(TestConfig)` — no env var mocking needed
- **Self-documenting**: the config class IS the list of all configuration
- **Inheritance**: `DevConfig(Config)` overrides only what differs

## Accessing Config in Application Code

Use `current_app.config` inside request context, or `app.config` if you have the app reference:

```python
from flask import current_app

def get_121_base_url():
    instance = current_app.config["INSTANCE_121"]
    return f"https://{instance}.121.global/api"
```

❌ Bad — direct `os.getenv()` in business logic:
```python
url = f'https://{os.getenv("INSTANCE121")}.121.global/api/users/login'
```

✅ Good — read from Flask config:
```python
url = f'https://{current_app.config["INSTANCE_121"]}.121.global/api/users/login'
```

✅ Best — inject config into service functions:
```python
# services/api_121.py
def login_121(username: str, password: str, *, base_url: str) -> str | None:
    """Login to 121 and return access token. Pure function, no Flask dependency."""
    url = f"{base_url}/users/login"
    resp = requests.post(url, data={"username": username, "password": password})
    if resp.ok:
        return resp.json()["access_token_general"]
    return None
```

This way the service function is testable without any Flask context.

## Environment Variable Naming

Use prefixed, uppercase names. Avoid generic names like `SECRETKEY`:

```dotenv
# ---------- Flask ----------
FLASK_APP=wsgi.py
FLASK_DEBUG=0

# ---------- App ----------
SECRET_KEY=                    # Flask session signing key
INSTANCE_121=                  # e.g. "training" or "production"

# ---------- Azure (if applicable) ----------
APPLICATIONINSIGHTS_CONNECTION_STRING=
```

## dotenv Support

Flask auto-loads `.flaskenv` and `.env` when `python-dotenv` is installed.

| File | Purpose | Git-tracked? |
|------|---------|--------------|
| `.flaskenv` | Flask CLI vars (`FLASK_APP`, `FLASK_DEBUG`) | ✅ Yes |
| `.env` | Application secrets (`SECRET_KEY`, API credentials) | ❌ No |
| `example.env` | Template showing all required vars with comments | ✅ Yes |

### example.env

Always provide an `example.env` with comments:

```dotenv
# ---------- Flask ----------
# FLASK_APP and FLASK_DEBUG are in .flaskenv (committed)

# ---------- App Secrets ----------
SECRET_KEY=                    # Generate with: python -c "import secrets; print(secrets.token_hex(32))"
INSTANCE_121=                  # 121 instance name (e.g. "training", "production-srilanka")

# ---------- Azure ----------
APPLICATIONINSIGHTS_CONNECTION_STRING=  # Optional: Azure App Insights
```

## Secrets in Production

For Azure App Service:

1. Store secrets as **Application Settings** in the Azure Portal (or via Terraform / Bicep)
2. They are injected as environment variables at runtime
3. For highly sensitive values, use **Azure Key Vault References**:
   ```
   @Microsoft.KeyVault(VaultName=my-vault;SecretName=secret-key)
   ```

**Never** commit secrets to git, even in "private" repos.
