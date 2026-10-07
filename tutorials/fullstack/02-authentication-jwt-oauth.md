# Full-Stack 02 — Authentication and Authorization: Sessions, JWT, OAuth 2.0, OIDC

## Why it matters
Named in the JD ("OAuth and JWT"). The AWS-specific implementation is in [../aws/06](../aws/06-cognito-auth-jwt-oauth.md). This tutorial is the vendor-neutral theory and the backend code.

## Vocabulary
- **Authentication (AuthN)**: who are you? **Authorization (AuthZ)**: what may you do?
- **Identity Provider (IdP)** / **Authorization Server (AS)**: issues tokens (Cognito, Auth0, Entra ID, Keycloak).
- **Resource Server**: your API. **Client**: the SPA/mobile/backend requesting access. **Resource Owner**: the user.

## Passwords (if you ever store them)
Hash with **Argon2id** or **bcrypt** (slow, salted, adaptive); never SHA-256 alone, never reversible encryption. Prefer delegating to an IdP so you never handle passwords. Add MFA, rate-limit login, generic error messages, breach-password checks.

## Sessions vs tokens

| | Server session (cookie) | JWT bearer token |
|---|---|---|
| State | Server stores session; cookie holds ID | Self-contained, signed claims; server stateless |
| Revocation | Instant (delete session) | Hard until expiry (short TTL, denylist, refresh rotation) |
| Scaling | Needs shared store (Redis) | Verify with public key anywhere |
| CSRF | Must defend (SameSite, tokens) | Not if sent in header (not auto-attached) |
| XSS | HttpOnly cookie hides it from JS | If in JS-accessible storage, stealable |
| Best for | Classic web apps, BFF pattern | APIs, microservices, mobile, third parties |

Good answer: "For a first-party SPA I'd lean toward the **BFF pattern** (cookie session with the SPA, tokens kept server-side); for APIs consumed by many clients, short-lived JWT access tokens plus rotating refresh tokens."

## JWT in depth

```
eyJhbGciOiJSUzI1NiIsImtpZCI6IjEyMyJ9 . eyJzdWIiOiJ1MSIsImV4cCI6MTc... . <signature>
  header: alg, typ, kid                    payload (claims)                  signature
```
Standard claims: `iss` (issuer), `sub` (subject/user id), `aud` (audience), `exp`, `nbf`, `iat`, `jti`. Custom: `scope`, `roles`.

- Signed (JWS), usually not encrypted: **anyone can read the payload**. No secrets/PII you wouldn't show the user.
- **HS256** (shared secret) vs **RS256/ES256** (private key signs, public key verifies via JWKS). Prefer asymmetric for distributed verification.
- **Verification checklist**: signature with a key chosen by `kid` from a trusted JWKS; **pin the allowed algorithms** (never accept `none`; never let the header pick HS256 with your public key as the secret); `iss`, `aud`, `exp`/`nbf` with small leeway; token type (`access` vs `id`).
- Keep access tokens short (5–15 min); refresh tokens long, **rotated on use**, revocable, stored safely.

### FastAPI: issue + verify (for a self-contained demo)

```python
# app/security.py
from datetime import datetime, timedelta, timezone
import jwt                                  # PyJWT
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

SECRET = "load-from-secrets-manager"       # HS256 demo only; use RS256 + IdP in production
ALGO = "HS256"
oauth2 = OAuth2PasswordBearer(tokenUrl="/auth/token")

def create_access_token(sub: str, minutes: int = 15) -> str:
    now = datetime.now(timezone.utc)
    return jwt.encode({"sub": sub, "iat": now, "exp": now + timedelta(minutes=minutes), "iss": "expense-api", "aud": "expense-web"},
                      SECRET, algorithm=ALGO)

def current_user_id(token: str = Depends(oauth2)) -> str:
    try:
        claims = jwt.decode(token, SECRET, algorithms=[ALGO], audience="expense-web", issuer="expense-api")
    except jwt.ExpiredSignatureError:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "token expired", headers={"WWW-Authenticate": "Bearer"})
    except jwt.PyJWTError:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "invalid token", headers={"WWW-Authenticate": "Bearer"})
    return claims["sub"]
```

### Gin (Go) middleware

```go
func AuthRequired(keyfunc jwt.Keyfunc, iss, aud string) gin.HandlerFunc {
	return func(c *gin.Context) {
		raw := strings.TrimPrefix(c.GetHeader("Authorization"), "Bearer ")
		tok, err := jwt.Parse(raw, keyfunc,
			jwt.WithValidMethods([]string{"RS256"}), jwt.WithIssuer(iss), jwt.WithAudience(aud), jwt.WithExpirationRequired())
		if err != nil || !tok.Valid {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"code": "unauthorized"})
			return
		}
		claims := tok.Claims.(jwt.MapClaims)
		c.Set("userID", claims["sub"])
		c.Next()
	}
}
```
(`github.com/golang-jwt/jwt/v5`; use `keyfunc` library or `MicahParks/keyfunc` to fetch JWKS.)

## OAuth 2.0 — delegated authorization

Roles: resource owner, client, authorization server, resource server. Grants:

| Grant | Use |
|---|---|
| **Authorization Code + PKCE** | Browser SPAs, mobile, and server apps; the default today |
| **Client Credentials** | Machine-to-machine, no user |
| **Refresh Token** | Get new access tokens |
| ~~Implicit~~ / ~~Password~~ | Deprecated; mention why (tokens in URL; apps handling raw passwords) |
| Device Code | TVs/CLIs |

**Scopes** = coarse permissions requested by the client (`expenses:read`, `expenses:write`). **PKCE**: client sends `code_challenge = SHA256(verifier)`, later proves possession with the `verifier`; defeats auth-code interception. Always validate `state` (CSRF) and exact `redirect_uri`.

**OIDC** (OpenID Connect) = identity layer on OAuth 2.0: adds the **ID token** (JWT about the user), `openid` scope, `/userinfo`, discovery (`/.well-known/openid-configuration`), JWKS.

> OAuth is not authentication by itself; "Sign in with X" uses OIDC.

## Authorization models
- **RBAC**: roles → permissions (`admin`, `member`). Simple; role explosion at scale.
- **ABAC / policy-based**: decide using attributes (user, resource, context). Tools: Cedar (AWS Verified Permissions), OPA.
- **Ownership / row-level**: `WHERE user_id = :sub` on every query; the most important check in this app.
- Enforce at **every layer**: API gateway (coarse), service (business rules), data layer (row-level).

### Prevent BOLA/IDOR (OWASP API #1)
```python
@router.get("/expenses/{expense_id}")
def get_expense(expense_id: UUID, user_id: str = Depends(current_user_id), db: Session = Depends(get_db)):
    e = db.scalar(select(Expense).where(Expense.id == expense_id, Expense.user_id == user_id))   # ownership in the query
    if e is None:
        raise HTTPException(404)               # same response whether missing or someone else's
    return e
```

## Web security essentials
- **XSS**: escape output (React does by default), CSP header, sanitize any HTML, avoid `dangerouslySetInnerHTML`.
- **CSRF**: matters for cookie auth → `SameSite=Lax/Strict`, CSRF tokens, check `Origin`. Bearer headers aren't auto-sent so are CSRF-immune.
- **CORS** is a browser policy, not security for your API; the API must still authenticate.
- **SQL injection**: parameterized queries.
- **Secrets**: never in git or the SPA bundle; Secrets Manager.
- **Transport**: HTTPS + HSTS. **Headers**: `X-Content-Type-Options`, `Content-Security-Policy`, `Referrer-Policy`.
- **Brute force**: rate limit, lockout/backoff, WAF.
- OWASP Top 10 and **OWASP API Security Top 10**: skim both before the interview.

## Exercise
1. Implement the JWT dependency/middleware in your backend and write tests: expired token → 401, wrong audience → 401, `alg: none` forged token → 401, user A requesting user B's expense → 404.
2. Decode a Cognito access token and list which claims you verify and why.
3. Draw the Authorization Code + PKCE sequence from memory.

## Interview Q&A
- **JWT downsides?** Hard revocation, size, easy to misuse (not encrypted, storage), algorithm confusion if verification is sloppy.
- **Where to store tokens in an SPA?** Memory + HttpOnly cookie refresh via BFF is safest; localStorage exposes to XSS.
- **ID token vs access token?** ID token = user identity for the client; access token = permission for APIs.
- **Why PKCE for SPAs?** Public clients can't keep a secret.
- **How do you rotate signing keys?** Publish new key in JWKS with new `kid`, keep old until tokens expire; verifiers cache and refetch on unknown `kid`.
- **401 or 403 for wrong user's resource?** 404 (or 403) to avoid leaking existence; never return data.
