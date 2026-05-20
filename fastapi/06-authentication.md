# 06 — Authentication

## API Key Authentication (Simple)

For internal services with one or two access levels:

```python
# dependencies.py
from fastapi import Depends, Security, HTTPException
from fastapi.security import APIKeyHeader
import secrets

from settings import AppSettings

api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

def get_settings() -> AppSettings:
    return AppSettings()

async def verify_api_key(
    api_key: str = Security(api_key_header),
    settings: AppSettings = Depends(get_settings),
) -> str:
    expected = settings.auth.api_key.get_secret_value()
    if not api_key or not secrets.compare_digest(api_key, expected):
        raise HTTPException(status_code=401, detail="Invalid or missing API key")
    return api_key

async def verify_write_key(
    api_key: str = Security(api_key_header),
    settings: AppSettings = Depends(get_settings),
) -> str:
    expected = settings.auth.api_key_write.get_secret_value()
    if not api_key or not secrets.compare_digest(api_key, expected):
        raise HTTPException(status_code=401, detail="Invalid or missing API key")
    return api_key
```

> **Note:** For quick prototypes, you may see `os.environ.get("API_KEY")` used directly. Prefer injecting settings via `Depends()` as shown above — it's consistent with [04-configuration-management.md](04-configuration-management.md) and easier to test.

Usage:
```python
@router.post("/search", dependencies=[Depends(verify_api_key)])
async def search(...): ...

@router.post("/create-vector-store", dependencies=[Depends(verify_write_key)])
async def create_vector_store(...): ...
```

## Bearer Token with Multi-Tenant Support (Production)

For APIs serving multiple organizations:

```python
# domain/models.py
from pydantic import BaseModel, SecretStr

class TenantApiKey(BaseModel):
    tenant_id: str
    key: SecretStr
    is_superuser: bool = False
```

```python
# settings.py — keys loaded from env (typically from Azure Key Vault)
class AuthSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="AUTH_")
    api_keys: list[TenantApiKey]  # JSON-encoded in AUTH_API_KEYS env var
```

```python
# dependencies.py
from fastapi import Request, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import secrets

security = HTTPBearer(auto_error=False)

async def authenticate_request(
    request: Request,
    credentials: HTTPAuthorizationCredentials = Security(security),
) -> TenantApiKey:
    if credentials is None:
        raise AuthenticationError("Authorization header required")

    token = credentials.credentials
    for tenant_key in request.app.state.api_keys:
        if secrets.compare_digest(token, tenant_key.key.get_secret_value()):
            return tenant_key

    raise AuthenticationError("Invalid API key")


def require_superuser(
    tenant: TenantApiKey = Depends(authenticate_request),
) -> TenantApiKey:
    if not tenant.is_superuser:
        raise AuthorizationError("Superuser access required")
    return tenant
```

Usage:
```python
@router.post("/v1/analyze")
async def analyze(
    body: AnalyzeRequest,
    tenant: TenantApiKey = Depends(authenticate_request),
): ...

@router.get("/v1/usage/all")
async def all_usage(
    tenant: TenantApiKey = Depends(require_superuser),
): ...
```

## Security Best Practices

### Always use constant-time comparison

```python
import secrets
# ✅ Prevents timing attacks
secrets.compare_digest(provided_key, expected_key)

# ❌ Vulnerable to timing attacks
provided_key == expected_key
```

### Never log credentials

```python
# ✅ Good — log tenant, not the key
logger.info("Request authenticated", extra={"tenant_id": tenant.tenant_id})

# ❌ Bad
logger.info(f"Authenticated with key: {api_key}")
```

### Use SecretStr for key storage

```python
class TenantApiKey(BaseModel):
    key: SecretStr  # repr shows '**********', not the actual value
```

### Store keys in Azure Key Vault in production

Not in app config or source code. Reference via App Service configuration:
```
AUTH_API_KEYS=@Microsoft.KeyVault(SecretUri=https://my-vault.vault.azure.net/secrets/api-keys)
```

## Webhook Authentication (Kobo/Twilio Pattern)

For endpoints that receive webhooks, the caller provides credentials to the target system:

```python
def required_headers_121(
    url121: str = Header(..., description="121 Platform URL"),
    username121: str = Header(..., description="121 username"),
    password121: str = Header(..., description="121 password"),
):
    return url121, username121, password121
```

This pattern is specific to middleware/connector services where the API itself doesn't store credentials — it just forwards them.