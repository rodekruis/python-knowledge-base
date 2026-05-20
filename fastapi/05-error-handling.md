# 05 — Error Handling

## Domain Exception Hierarchy

Define a hierarchy of domain exceptions that are independent of HTTP:

```python
# domain/errors.py

class DomainError(Exception):
    """Base class for all domain errors."""
    pass

class AuthenticationError(DomainError):
    """Credentials missing or invalid."""
    pass

class AuthorizationError(DomainError):
    """Authenticated but not permitted."""
    pass

class NotFoundError(DomainError):
    """Requested resource does not exist."""
    pass

class ValidationError(DomainError):
    """Input violates business rules."""
    pass

class ExternalServiceError(DomainError):
    """An external dependency failed."""
    pass

class TimeoutError(ExternalServiceError):
    """An external call timed out."""
    pass

class RateLimitError(ExternalServiceError):
    """External service rate-limited us."""
    pass
```

## Mapping Exceptions to HTTP Responses

Register global exception handlers that map domain errors to consistent JSON responses:

```python
# api/app.py
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

def register_exception_handlers(app: FastAPI):
    @app.exception_handler(AuthenticationError)
    async def auth_error(request: Request, exc: AuthenticationError):
        return _error_response(401, "authentication_required", str(exc), request)

    @app.exception_handler(AuthorizationError)
    async def authz_error(request: Request, exc: AuthorizationError):
        return _error_response(403, "forbidden", str(exc), request)

    @app.exception_handler(NotFoundError)
    async def not_found(request: Request, exc: NotFoundError):
        return _error_response(404, "not_found", str(exc), request)

    @app.exception_handler(ValidationError)
    async def validation_error(request: Request, exc: ValidationError):
        return _error_response(422, "validation_error", str(exc), request)

    @app.exception_handler(RequestValidationError)
    async def pydantic_error(request: Request, exc: RequestValidationError):
        return _error_response(422, "validation_error", str(exc), request)

    @app.exception_handler(ExternalServiceError)
    async def external_error(request: Request, exc: ExternalServiceError):
        return _error_response(502, "external_service_error", str(exc), request)

    @app.exception_handler(Exception)
    async def catch_all(request: Request, exc: Exception):
        logger.exception("Unhandled exception", extra={"request_id": _get_request_id(request)})
        return _error_response(500, "internal_error", "An unexpected error occurred", request)


def _error_response(status: int, code: str, message: str, request: Request) -> JSONResponse:
    request_id = request.state.request_id if hasattr(request.state, "request_id") else None
    return JSONResponse(
        status_code=status,
        content={
            "error": {
                "code": code,
                "message": message,
                "request_id": request_id,
            }
        },
    )
```

## Standard Error Response Schema

All error responses should follow the same shape:

```json
{
  "error": {
    "code": "authentication_required",
    "message": "Missing or invalid API key",
    "request_id": "req_abc123..."
  }
}
```

Document this in your Pydantic schemas:

```python
class ApiErrorDetail(BaseModel):
    code: str
    message: str
    request_id: str | None = None

class ApiErrorResponse(BaseModel):
    error: ApiErrorDetail
```

## Error Handling in Adapters

Wrap external service calls with try/except and translate to domain exceptions:

```python
# adapters/llm_client.py
async def generate(self, prompt: str) -> str:
    try:
        response = await self.client.chat.completions.create(...)
        return response.choices[0].message.content
    except openai.RateLimitError as e:
        raise RateLimitError(f"LLM rate limited: {e}") from e
    except openai.APITimeoutError as e:
        raise TimeoutError(f"LLM timed out: {e}") from e
    except openai.APIError as e:
        raise ExternalServiceError(f"LLM error: {e}") from e
```

```python
# utils/translator.py
def translate(from_lang: str, to_lang: str, text: str) -> str:
    try:
        response = requests.post(url, json=body, headers=headers)
        response.raise_for_status()
        return response.json()[0]["translations"][0]["text"]
    except requests.RequestException as e:
        logger.error(f"Translation failed: {e}")
        return text  # graceful fallback
```

## Guidelines

1. **Never use bare `except Exception`** — always catch specific exceptions
2. **Never return AND raise** — pick one error reporting pattern per function
3. **Include `from e`** when re-raising — preserves the traceback chain
4. **Log at the boundary** — log errors in exception handlers, not deep in business logic
5. **Consumers code against `error.code` strings** — never against HTTP status alone
6. **Unhandled exceptions → 500** with generic message (never leak internals to clients)

