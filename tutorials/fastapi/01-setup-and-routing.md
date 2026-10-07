# FastAPI 01 — Setup, Routing and Project Layout

## Why FastAPI
The JD asks for "Python or Go". FastAPI gives: type-hint-driven validation (Pydantic), automatic OpenAPI docs, async support, dependency injection, great performance (Starlette + uvicorn). Pair it with Go/Gin ([../gin/](../gin/)) and compare.

## Setup

```bash
mkdir api-fastapi && cd api-fastapi
python3.12 -m venv .venv && source .venv/bin/activate
pip install "fastapi[standard]" "sqlalchemy>=2" "psycopg[binary]" alembic pydantic-settings "pyjwt[crypto]" httpx pytest
pip freeze > requirements.txt          # or use uv / poetry
fastapi dev app/main.py                # hot reload; docs at /docs (Swagger) and /redoc
```

## Layout

> **Start from the clean-architecture layout in [07](07-clean-architecture.md)** (`domain / application / adapters / interface / container.py`). The flat layout below is the simplest starting point for learning FastAPI mechanics; tutorials 01–05 explain the framework pieces, and 07 shows where each piece belongs (routers and schemas in `interface/http`, repositories in `adapters/persistence`, rules in `domain`, orchestration in `application`).

```
api-fastapi/
  app/
    main.py            # app factory, middleware, routers
    config.py          # settings from env
    deps.py            # shared dependencies (db session, current user)
    models.py          # SQLAlchemy ORM models
    schemas.py         # Pydantic request/response models
    routers/expenses.py
    services/expenses.py
    repositories/expenses.py
  alembic/             # migrations
  tests/
  Dockerfile
```

## Hello, typed API

```python
# app/main.py
from fastapi import FastAPI
from app.routers import expenses

app = FastAPI(title="Expense Tracker API", version="1.0.0")
app.include_router(expenses.router)

@app.get("/health", tags=["ops"])
def health() -> dict[str, str]:
    return {"status": "ok"}
```

## Routing with path, query, body

```python
# app/routers/expenses.py
from datetime import date
from typing import Annotated
from uuid import UUID
from fastapi import APIRouter, Depends, HTTPException, Query, Response, status
from app.schemas import ExpenseCreate, ExpenseOut, ExpensePage, ExpenseUpdate

router = APIRouter(prefix="/expenses", tags=["expenses"])

@router.get("", response_model=ExpensePage)
def list_expenses(
    start: Annotated[date | None, Query(alias="from")] = None,
    end: Annotated[date | None, Query(alias="to")] = None,
    category: str | None = None,
    limit: Annotated[int, Query(ge=1, le=100)] = 50,
    cursor: str | None = None,
):
    ...

@router.post("", response_model=ExpenseOut, status_code=status.HTTP_201_CREATED)
def create_expense(body: ExpenseCreate, response: Response):
    created = ...
    response.headers["Location"] = f"/expenses/{created.id}"
    return created

@router.get("/{expense_id}", response_model=ExpenseOut)
def get_expense(expense_id: UUID):
    ...

@router.patch("/{expense_id}", response_model=ExpenseOut)
def update_expense(expense_id: UUID, body: ExpenseUpdate):
    ...

@router.delete("/{expense_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_expense(expense_id: UUID):
    ...
```

How FastAPI decides where a parameter comes from: name in the path → path param; Pydantic model → JSON body; simple type → query param; `Header()`, `Cookie()`, `Form()`, `File()` explicit.

## `async def` vs `def`
- `async def` handlers run on the event loop: use only with **async-native libraries** (httpx, asyncpg, aioboto3). A blocking call inside blocks *all* requests.
- Plain `def` handlers run in a **threadpool**: safe for blocking libs (psycopg sync, boto3).
- Rule: if you call blocking code, use `def` (or `run_in_threadpool`); never `time.sleep`/`requests` inside `async def`.
- CPU-bound work → separate process/worker, not the event loop.

## Errors

```python
class DomainError(Exception):
    def __init__(self, code: str, message: str, status_code: int = 422):
        self.code, self.message, self.status_code = code, message, status_code

@app.exception_handler(DomainError)
async def domain_error_handler(request, exc: DomainError):
    return JSONResponse(
        status_code=exc.status_code,
        media_type="application/problem+json",
        content={"type": f"https://api.example.com/problems/{exc.code}", "title": exc.message,
                 "status": exc.status_code, "instance": str(request.url.path)},
    )
```
`HTTPException(status_code=404, detail="…")` for quick cases; `RequestValidationError` (422) is raised automatically on bad input and can be customized.

## Middleware, CORS, lifespan

```python
from contextlib import asynccontextmanager
from fastapi.middleware.cors import CORSMiddleware
import time, uuid, logging

@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup: create pools, load JWKS, warm caches
    yield
    # shutdown: close pools

app = FastAPI(lifespan=lifespan)
app.add_middleware(CORSMiddleware, allow_origins=["http://localhost:5173", "https://app.example.com"],
                   allow_methods=["*"], allow_headers=["*"], allow_credentials=False)

@app.middleware("http")
async def request_context(request, call_next):
    rid = request.headers.get("x-request-id", str(uuid.uuid4()))
    start = time.perf_counter()
    response = await call_next(request)
    response.headers["x-request-id"] = rid
    logging.info("request", extra={"rid": rid, "path": request.url.path, "status": response.status_code,
                                   "ms": round((time.perf_counter() - start) * 1000)})
    return response
```

## Settings

```python
# app/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env")
    database_url: str
    cognito_region: str = "us-east-1"
    cognito_pool_id: str
    cognito_client_id: str
    bedrock_model_id: str | None = None

settings = Settings()
```
Config from env (12-factor); in AWS inject via task definition secrets/Lambda env.

## Exercise
Create the app skeleton, the six endpoints returning stub data, and verify `/docs`. Export the OpenAPI: `curl localhost:8000/openapi.json > openapi.json` and generate TS types in the React app.

## Interview Q&A
- **Why is FastAPI fast?** ASGI (uvicorn/Starlette), async I/O, Pydantic v2 core in Rust; not magic. Blocking code kills it.
- **`def` vs `async def` handler?** See above.
- **Where does the OpenAPI come from?** Introspection of type hints/Pydantic models/route metadata.
- **WSGI vs ASGI?** Sync request/response vs async protocol supporting websockets, streaming, HTTP/2.
