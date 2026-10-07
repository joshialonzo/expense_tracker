# AWS 06 — Authentication and Authorization with Cognito (OAuth 2.0, OIDC, JWT)

## Why it matters
The JD explicitly asks for OAuth and JWT. Cognito is the AWS-native way to demonstrate them. Concepts here are portable (Auth0, Entra ID, Keycloak). General theory is in [../fullstack/02-authentication-jwt-oauth.md](../fullstack/02-authentication-jwt-oauth.md).

## Cognito pieces

| Piece | Purpose |
|---|---|
| **User pool** | User directory + sign-up/sign-in + token issuer (an OIDC provider). |
| **App client** | Your SPA/app registration (client id; no secret for public SPAs). |
| **Hosted UI / managed login** | Prebuilt login pages, social/SAML federation. |
| **Identity pool** | Exchanges tokens for temporary **AWS credentials** (to call S3 directly). Different from a user pool! |

User pool = *authentication* ("who are you", tokens). Identity pool = *AWS credentials* for that identity.

## Tokens you get
- **ID token** — who the user is (claims: `sub`, `email`). For the client app.
- **Access token** — what the client may call (`scope`, `client_id`, `cognito:groups`). Send this to your API.
- **Refresh token** — long-lived, gets new ID/access tokens. Store carefully; rotate/revoke.

JWT structure: `header.payload.signature` (base64url). Header has `alg` + `kid` (which key signed). Payload has `iss`, `sub`, `aud`/`client_id`, `exp`, `iat`, `token_use`. **A JWT is signed, not encrypted** — never put secrets in it.

## The right flow for a SPA: Authorization Code + PKCE

```
1. SPA generates code_verifier (random) → code_challenge = BASE64URL(SHA256(verifier))
2. Redirect to  /oauth2/authorize?response_type=code&client_id=…&redirect_uri=…
                &scope=openid+email&code_challenge=…&code_challenge_method=S256&state=…
3. User signs in on Cognito's Hosted UI
4. Redirect back with ?code=…&state=…   (verify state → CSRF protection)
5. SPA POSTs to /oauth2/token with code + code_verifier → tokens
6. SPA calls API with  Authorization: Bearer <access_token>
```
Implicit flow (tokens in the URL fragment) is deprecated. PKCE replaces the client secret that a browser can't keep.

## Frontend: Amplify Auth (simplest)

```ts
// npm i aws-amplify
import { Amplify } from "aws-amplify";
import { signInWithRedirect, signOut, fetchAuthSession } from "aws-amplify/auth";

Amplify.configure({
  Auth: {
    Cognito: {
      userPoolId: import.meta.env.VITE_USER_POOL_ID,
      userPoolClientId: import.meta.env.VITE_USER_POOL_CLIENT_ID,
      loginWith: {
        oauth: {
          domain: import.meta.env.VITE_COGNITO_DOMAIN,
          scopes: ["openid", "email"],
          redirectSignIn: ["http://localhost:5173/"],
          redirectSignOut: ["http://localhost:5173/"],
          responseType: "code",
        },
      },
    },
  },
});

export async function authHeader(): Promise<Record<string, string>> {
  const { tokens } = await fetchAuthSession();      // auto-refreshes
  const token = tokens?.accessToken?.toString();
  return token ? { Authorization: `Bearer ${token}` } : {};
}
```

(You can also use `oidc-client-ts` / `react-oidc-context` — vendor-neutral and a good thing to mention.)

## Backend: validate the token

### Option A — API Gateway JWT authorizer (no code)
Set issuer `https://cognito-idp.<region>.amazonaws.com/<poolId>` and audience = app client id. For Cognito *access* tokens there is no `aud`, instead `client_id`; the HTTP API JWT authorizer accepts `client_id` for the audience check. Authorized scopes can be required per route. The Lambda receives verified claims in `event.requestContext.authorizer.jwt.claims`.

### Option B — Verify in your own service (FastAPI example)

```python
import httpx, jwt                       # PyJWT[crypto]
from functools import lru_cache
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer

REGION, POOL, CLIENT = "us-east-1", "us-east-1_XXXX", "abc123clientid"
ISSUER = f"https://cognito-idp.{REGION}.amazonaws.com/{POOL}"
bearer = HTTPBearer()

@lru_cache
def jwks_client() -> jwt.PyJWKClient:
    return jwt.PyJWKClient(f"{ISSUER}/.well-known/jwks.json")   # caches keys

def current_user(creds=Depends(bearer)) -> dict:
    token = creds.credentials
    try:
        key = jwks_client().get_signing_key_from_jwt(token).key
        claims = jwt.decode(token, key, algorithms=["RS256"], issuer=ISSUER,
                            options={"verify_aud": False})   # access token: check client_id manually
    except jwt.PyJWTError:
        raise HTTPException(401, "invalid token")
    if claims.get("token_use") != "access" or claims.get("client_id") != CLIENT:
        raise HTTPException(401, "wrong token")
    return claims
```
Validate: signature (via JWKS, by `kid`), `alg` pinned to RS256 (never trust the header's alg blindly; reject `none`), `iss`, `exp`, `token_use`, audience/`client_id`.

## Authorization (after authentication)
- **Coarse**: scopes / Cognito groups (`cognito:groups: ["admin"]`) → RBAC.
- **Fine / ownership**: every query filters by `claims["sub"]`. `GET /expenses/{id}` must return 404 unless `expense.user_id == sub` (prevent **IDOR / BOLA**, the #1 API vulnerability).
- Never accept `userId` from the request body.

## Token storage in the SPA (a favorite question)
- `localStorage`: simple but readable by any XSS.
- In-memory + refresh token in an **HttpOnly, Secure, SameSite cookie** (via a backend-for-frontend): XSS-resistant but needs CSRF protection.
- Mitigate XSS regardless: CSP, no `dangerouslySetInnerHTML` with untrusted data, dependency hygiene. Keep access tokens short-lived (5–60 min).

## Service-to-service: client credentials
Machine-to-machine (e.g., a report batch job calling the API): OAuth **client_credentials** grant with a resource server and custom scopes in Cognito. Or use IAM SigV4 with IAM authorization on API Gateway for AWS-internal callers.

## Exercise
Create a user pool + app client + domain, attach a JWT authorizer to an HTTP API route, sign in from the React app, call the route, then decode the token on jwt.io and explain every claim. Finally, tamper with the payload and show the 401.

## Interview Q&A
- **JWT vs session cookie?** JWT = stateless verification, good for APIs/microservices, hard to revoke before expiry; sessions = server state, instant revocation. Hybrid: short access JWT + rotating refresh.
- **OAuth vs OIDC?** OAuth 2.0 = delegated *authorization*; OIDC adds an identity layer (ID token, `userinfo`, discovery).
- **Why PKCE?** Binds the auth code to the client that started the flow; prevents code interception for public clients.
- **How do you revoke a JWT?** Short expiry, refresh token revocation (Cognito `GlobalSignOut`/`RevokeToken`), denylist by `jti` if essential.
- **ID token vs access token at the API?** Send the access token; ID token is for the client.
- **What is JWKS?** Public keys endpoint; verifier picks key by `kid`; cache it and handle rotation.
