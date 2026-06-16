# 03 — Routes and Schemas

## Router Organization

Use **[`APIRouter`](README.md#glossary)** with tags and include with optional prefixes:

```python
# routes/chat.py
from fastapi import APIRouter

router = APIRouter()

@router.post("/chat", tags=["chat"])
async def chat(...):
    ...
```

```python
# main.py or app.py
app.include_router(chat_router)
app.include_router(search_router)
# Or with prefix for versioning:
app.include_router(chat_router, prefix="/v1")
```

### Naming Conventions

- File: `routes/<resource>.py` or `routes/routes_<integration>.py`
- Router variable: `router = APIRouter()`
- Route functions: use verb-noun (`create_vector_store`, `search_documents`)
- Tags: one tag per logical group, used in the OpenAPI UI

## Dependency Injection

FastAPI's **[`Depends()`](README.md#glossary)** is the primary DI mechanism. Use it for:

### Authentication

```python
# dependencies.py
from fastapi import Depends, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer(auto_error=False)

async def authenticate_request(
    request: Request,
    credentials: HTTPAuthorizationCredentials = Security(security),
) -> TenantApiKey:
    if credentials is None:
        raise AuthenticationError("Missing authorization header")
    # constant-time compare
    ...
```

### Service injection (from app.state)

```python
def get_orchestrator(request: Request) -> Orchestrator:
    return request.app.state.orchestrator
```

### Header-based credentials (for webhook integrations)

```python
def required_headers_espocrm(
    targeturl: str = Header(..., description="EspoCRM instance URL"),
    targetkey: str = Header(..., description="EspoCRM API key"),
):
    return targeturl, targetkey

@router.post("/kobo-to-espocrm", tags=["EspoCRM"])
async def kobo_to_espo(request: Request, deps=Depends(required_headers_espocrm)):
    url, key = deps
    ...
```

## Pydantic Request/Response Schemas

Use **[Pydantic models](README.md#glossary)** to define all request and response shapes.

### Define explicit schemas

Always define request and response models — never use raw `dict` or `Request.json()`:

```python
# schemas.py
from pydantic import BaseModel, Field

class SearchRequest(BaseModel):
    query: str = Field(..., description="The search query text")
    google_sheet_id: str = Field(..., alias="googleSheetId")
    lang: str = Field(default="en", description="ISO 639-1 language code")
    k: int = Field(default=5, ge=1, le=20, description="Number of results")

class SearchResult(BaseModel):
    question: str
    answer: str
    score: float
    category_id: int = Field(alias="categoryID")

class SearchResponse(BaseModel):
    results: list[SearchResult]
```

### Use response_model for documentation and validation

```python
@router.post("/search", response_model=SearchResponse, tags=["search"])
async def search(payload: SearchRequest) -> SearchResponse:
    ...
```

### Separate API schemas from domain models

The API schema is your contract with external consumers. Domain models are your internal representation. Never expose domain internals directly:

> This split exists because the HTTP layer is just one *adapter* into the application. See [hexagonal architecture](01-project-structure.md#key-principles) for the underlying rationale.

```python
# Route handler maps between them
@router.post("/v1/analyze", response_model=ApiAnalyzeResponse)
async def analyze(
    body: ApiAnalyzeRequest,
    orchestrator: Orchestrator = Depends(get_orchestrator),
) -> ApiAnalyzeResponse:
    # Map API schema → domain
    request = AnalysisRequest(
        documents=tuple(body.documents),
        language=Language(body.language),
    )
    # Call business logic
    result = await orchestrator.analyze(request)
    # Map domain → API schema
    return ApiAnalyzeResponse.from_domain(result)
```

## Route Handler Guidelines

### Keep handlers thin

Route handlers should only:
1. Parse/validate input (done by FastAPI + Pydantic)
2. Call a service/orchestrator
3. Map the result to a response schema

```python
# ✅ Good — thin handler
@router.post("/chat", response_model=ChatResponse, tags=["chat"])
async def chat(payload: ChatRequest, agent=Depends(get_agent)):
    result = agent.invoke(payload.message, thread_id=payload.thread_id)
    return ChatResponse(response=result.text, context=result.sources)
```

```python
# ❌ Bad — business logic in the handler
@router.post("/chat", tags=["chat"])
async def chat(payload: ChatRequest):
    # 50 lines of translation, retrieval, prompting, etc.
    ...
```

### Always declare status codes and response types

```python
@router.post("/create-vector-store", status_code=201, response_model=CreateResponse)
@router.delete("/delete-vector-store", status_code=200)
```

## Query Parameters vs Body vs Path

| Use | When |
|-----|------|
| **Path** (`/items/{id}`) | Identifying a specific resource |
| **Query** (`?lang=en&k=5`) | Filtering, pagination, optional modifiers |
| **Body** (JSON) | Complex payloads, creating/updating resources |

