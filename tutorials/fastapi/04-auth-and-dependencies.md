# FastAPI 04 — Dependency Injection, JWT Auth (Cognito) and Authorization

## Dependency injection

> **Clean-architecture note:** `Depends()` is great for *HTTP-level* concerns (identity, request parsing). Business collaborators (repositories, clock, LLM) are wired in the **composition root** (`container.py`) and handed to use cases by constructor, as in [07](07-clean-architecture.md). Don't scatter `Depends(get_db)` through use cases.

`Depends()` is FastAPI's DI: declare what a handler needs; the framework builds it per request, caches within the request, and supports overriding in tests.

```python
def get_db() -> Iterator[Session]: ...
def get_settings() -> Settings: return settings
def get_repo(db=Depends(get_db), user_id=Depends(current_user_uuid)) -> ExpenseRepo: ...

@router.get("/{expense_id}", response_model=ExpenseOut)
def get_expense(expense_id: UUID, repo: ExpenseRepo = Depends(get_repo)):
    e = repo.get(expense_id)
    if e is None:
        raise HTTPException(404, "expense not found")
    return e
```
Router-wide dependencies: `APIRouter(dependencies=[Depends(require_auth)])`. App-wide: `FastAPI(dependencies=[...])`.

## Validating Cognito JWTs

```python
# app/auth.py
from functools import lru_cache
from typing import Annotated
import jwt
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
from sqlalchemy import select
from sqlalchemy.orm import Session
from app.config import settings
from app.db import get_db
from app.models import User

ISSUER = f"https://cognito-idp.{settings.cognito_region}.amazonaws.com/{settings.cognito_pool_id}"
bearer = HTTPBearer(auto_error=False)

@lru_cache
def _jwks() -> jwt.PyJWKClient:
    return jwt.PyJWKClient(f"{ISSUER}/.well-known/jwks.json", cache_keys=True, lifespan=3600)

class Claims(dict):
    @property
    def sub(self) -> str: return self["sub"]
    @property
    def groups(self) -> list[str]: return self.get("cognito:groups", [])

def current_claims(creds: Annotated[HTTPAuthorizationCredentials | None, Depends(bearer)]) -> Claims:
    unauthorized = HTTPException(status.HTTP_401_UNAUTHORIZED, "invalid or missing token",
                                 headers={"WWW-Authenticate": "Bearer"})
    if creds is None:
        raise unauthorized
    try:
        key = _jwks().get_signing_key_from_jwt(creds.credentials).key
        claims = jwt.decode(creds.credentials, key, algorithms=["RS256"], issuer=ISSUER,
                            options={"require": ["exp", "iss", "sub"], "verify_aud": False})
    except jwt.PyJWTError:
        raise unauthorized
    if claims.get("token_use") != "access" or claims.get("client_id") != settings.cognito_client_id:
        raise unauthorized
    return Claims(claims)

def current_user(claims: Claims = Depends(current_claims), db: Session = Depends(get_db)) -> User:
    """Just-in-time provisioning: first valid token creates the local user row."""
    user = db.scalar(select(User).where(User.cognito_sub == claims.sub))
    if user is None:
        user = User(cognito_sub=claims.sub, email=claims.get("email", f"{claims.sub}@unknown"))
        db.add(user); db.commit()
    return user

def current_user_uuid(user: User = Depends(current_user)):
    return user.id

def require_group(group: str):
    def checker(claims: Claims = Depends(current_claims)):
        if group not in claims.groups:
            raise HTTPException(status.HTTP_403_FORBIDDEN, "insufficient permissions")
    return Depends(checker)

# usage:
@router.get("/admin/stats", dependencies=[require_group("admin")])
def stats(): ...
```
(Access tokens from Cognito have no email; the ID token does. Either call `/userinfo`, or add a Pre Token Generation Lambda to include it, or store only `sub`.)

## Scopes with `Security`

```python
from fastapi import Security
from fastapi.security import SecurityScopes

def require_scopes(security_scopes: SecurityScopes, claims: Claims = Depends(current_claims)):
    granted = set(claims.get("scope", "").split())
    if not set(security_scopes.scopes) <= granted:
        raise HTTPException(403, "missing scope")

@router.delete("/{id}", dependencies=[Security(require_scopes, scopes=["expenses:write"])])
def delete_expense(...): ...
```

## Make Swagger UI use your auth
Using `HTTPBearer` adds an "Authorize" button where you paste a token, which is handy for manual testing against Cognito.

## Testing auth

```python
import time, jwt
from cryptography.hazmat.primitives.asymmetric import rsa

def make_token(private_key, **overrides):
    now = int(time.time())
    claims = {"sub": "u1", "iss": ISSUER, "exp": now + 300, "iat": now, "token_use": "access",
              "client_id": settings.cognito_client_id, **overrides}
    return jwt.encode(claims, private_key, algorithm="RS256", headers={"kid": "test"})

def test_expired_token_rejected(client, private_key):
    t = make_token(private_key, exp=int(time.time()) - 10)
    assert client.get("/expenses", headers={"Authorization": f"Bearer {t}"}).status_code == 401
```
In unit tests override `current_claims` via `app.dependency_overrides`; keep a few tests that exercise real verification with a locally generated key and a patched JWKS client.

## Other production concerns
- **Rate limiting**: API Gateway/WAF, or `slowapi`/Redis in-app.
- **Security headers** via middleware; **HTTPS** terminated at ALB/CloudFront/API Gateway.
- **Audit logging** for sensitive actions (delete, export).
- **Secrets** from Secrets Manager (`boto3` at startup) not committed.

## Exercise
Implement `current_claims`, JIT user provisioning, `require_group`, and the 6 auth tests: no token, malformed, expired, wrong issuer, wrong `client_id`, valid. Add an ownership test (user A → user B's expense → 404).

## Interview Q&A
- **How does `Depends` caching work?** Same dependency is evaluated once per request; `use_cache=False` to disable.
- **Why verify `client_id`/`token_use`?** Avoid accepting ID tokens or tokens issued to other apps in the same pool.
- **Where does authorization live?** Edge for coarse checks (scopes/groups); repository/service for ownership.
- **How do you override deps in tests?** `app.dependency_overrides[dep] = fake`.
