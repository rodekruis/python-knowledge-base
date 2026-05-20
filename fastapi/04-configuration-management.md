# 04 — Configuration Management

## Use pydantic-settings

Never scatter `os.environ[...]` calls throughout your code. Centralize all configuration using **[pydantic-settings (`BaseSettings`)](README.md#glossary)**:

```python
# settings.py
from pydantic import Field, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict

class LLMSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="LLM_")

    model: str = "gpt-4o"
    api_key: SecretStr                    # required, no default → fails fast
    endpoint: str = ""
    api_version: str = "2024-06-01"
    timeout_seconds: float = 60.0

class DatabaseSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="DB_")

    host: str = "localhost"
    port: int = 5432
    user: str = ""
    password: SecretStr = SecretStr("")
    name: str = "app"

    @property
    def url(self) -> str:
        pwd = self.password.get_secret_value()
        return f"postgresql://{self.user}:{pwd}@{self.host}:{self.port}/{self.name}"

class AuthSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="AUTH_")

    api_key: SecretStr                    # read-only endpoints
    api_key_write: SecretStr = SecretStr("")  # write endpoints

class AppSettings(BaseSettings):
    """Composed root settings — instantiate once at startup."""

    llm: LLMSettings = Field(default_factory=LLMSettings)
    db: DatabaseSettings = Field(default_factory=DatabaseSettings)
    auth: AuthSettings = Field(default_factory=AuthSettings)
    port: int = 8000
    environment: str = "development"
```

### Benefits

- **Fails fast**: required fields without defaults crash immediately at startup (not when the first request hits)
- **Type-safe**: `int`, `SecretStr`, `bool` are validated automatically
- **Testable**: override via constructor in tests: `LLMSettings(api_key="test")`
- **Documented**: the class definition IS the documentation of all env vars
- **Namespaced**: `env_prefix` prevents collisions (`LLM_MODEL`, `DB_HOST`, etc.)

## Environment Variable Naming

Use prefixed, uppercase names with underscores:

```
LLM_MODEL=gpt-4o
LLM_API_KEY=sk-...
LLM_ENDPOINT=https://my-openai.azure.com/

DB_HOST=my-server.postgres.database.azure.com
DB_USER=appuser
DB_PASSWORD=...

AUTH_API_KEY=...
AUTH_API_KEY_WRITE=...

APPLICATIONINSIGHTS_CONNECTION_STRING=...
```

## dotenv Support

For local development, use a `.env` file loaded by pydantic-settings or python-dotenv:

```python
# If using pydantic-settings:
class AppSettings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")
```

Or explicitly:
```python
# main.py
from dotenv import load_dotenv
load_dotenv()
```

## example.env

Always provide an `example.env` with comments explaining each variable:

```env
# ---------- App ----------
PORT=8000

# ---------- Authentication ----------
AUTH_API_KEY=              # Read-only endpoints
AUTH_API_KEY_WRITE=        # Write endpoints (create/delete)

# ---------- Azure OpenAI ----------
LLM_MODEL=                # e.g. gpt-4o
LLM_API_KEY=
LLM_ENDPOINT=             # e.g. https://my-instance.openai.azure.com/
LLM_API_VERSION=2024-06-01

# ---------- Database ----------
DB_HOST=
DB_USER=
DB_PASSWORD=
```

## Secrets in Production

- **Never commit secrets** — `.env` must be in `.gitignore`
- **Azure Key Vault** for production secrets, referenced via App Service configuration
- Use **[`SecretStr`](README.md#glossary)** for any credential field — prevents accidental logging:

```python
class MySettings(BaseSettings):
    api_key: SecretStr

settings = MySettings()
print(settings.api_key)                    # SecretStr('**********')
print(settings.api_key.get_secret_value()) # actual value (use sparingly)
```

## Loading Order

For simpler apps (hia-search pattern):

```python
# main.py
from dotenv import load_dotenv
load_dotenv()  # loads .env BEFORE any os.environ access
```

For larger apps (QFA pattern):

```python
# Pydantic-settings handles .env automatically via model_config
# No explicit load_dotenv() needed
settings = AppSettings()  # reads env + .env file
```