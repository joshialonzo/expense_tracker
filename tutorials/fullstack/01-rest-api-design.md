# Full-Stack 01 — RESTful API Design Best Practices

## Why it matters
"Design and implement RESTful APIs using industry best practices" is a JD line. Expect: "design the API for X", "what status code for Y", "how do you version/paginate/make it idempotent".

## REST in one paragraph
Resources identified by URLs, manipulated through HTTP methods with standard semantics, stateless requests, representations (JSON), cacheable, uniform interface. Most "REST" APIs are really "resource-oriented HTTP/JSON APIs"; it's fine to say so.

## Resource modeling
- **Nouns, plural, hierarchical only when ownership is real**: `/expenses`, `/expenses/{id}`, `/expenses/{id}/receipt`. Avoid verbs (`/getExpenses`).
- Identity of the *current user* comes from the token: `/me/expenses` or just `/expenses` scoped by `sub`, never `/users/{id}/expenses` if the caller can only see their own (otherwise you invite IDOR).
- Actions that aren't CRUD: model as a sub-resource or a verb-ish noun: `POST /expenses/{id}/receipt-upload-url`, `POST /reports/exports`.

## Methods and semantics

| Method | Use | Safe | Idempotent | Typical success |
|---|---|---|---|---|
| GET | Read | ✔ | ✔ | 200 |
| POST | Create / non-idempotent action | ✘ | ✘ | 201 + `Location` header, or 202 for async |
| PUT | Replace whole resource | ✘ | ✔ | 200/204 |
| PATCH | Partial update | ✘ | not guaranteed | 200/204 |
| DELETE | Remove | ✘ | ✔ | 204 |

## Status codes you must use correctly
- **200** OK, **201** Created (+ `Location`), **202** Accepted (async), **204** No Content
- **400** malformed request, **401** unauthenticated (missing/invalid token), **403** authenticated but not allowed, **404** not found (also use for "exists but not yours" to avoid leaking existence), **409** conflict (duplicate, version mismatch), **412/428** preconditions (ETag / If-Match), **415** unsupported media type, **422** semantic validation error, **429** rate limited (+ `Retry-After`)
- **500** server bug, **502/503/504** upstream/unavailable/timeout

Consistent **error body** (RFC 9457 Problem Details is the standard):

```json
{
  "type": "https://api.example.com/problems/validation-error",
  "title": "Validation failed",
  "status": 422,
  "detail": "amountCents must be greater than 0",
  "instance": "/expenses",
  "errors": [{ "field": "amountCents", "message": "must be > 0" }],
  "traceId": "3f9a-..."
}
```
Content type `application/problem+json`. Include a trace/request ID so support can find logs.

## Request/response design
- **JSON field naming**: pick one (`camelCase` for JS-friendly) and stay consistent.
- **Money**: integer minor units + currency code (`{"amountCents":1250,"currency":"USD"}`); never floats.
- **Dates**: ISO 8601 UTC (`2026-10-06T14:03:00Z`); dates without time as `YYYY-MM-DD`.
- **IDs**: opaque strings (UUID/ULID). ULID/UUIDv7 are sortable by time.
- **Don't leak internals**: separate DTOs from DB models; whitelist fields (prevents mass assignment; e.g. a client must not set `userId` or `isAdmin`).
- **PATCH**: JSON Merge Patch (`application/merge-patch+json`) is simple; `null` clears a field.

## Collections: filtering, sorting, pagination

```
GET /expenses?from=2026-10-01&to=2026-10-31&category=food&sort=-date&limit=50&cursor=eyJkIjoiMjAyNi0xMC0wMiIsImlkIjoiZTkifQ
```
Response:

```json
{ "items": [ ... ], "nextCursor": "eyJkIjoi...", "total": null }
```

| Strategy | Pros | Cons |
|---|---|---|
| **Offset/limit** (`?page=3&size=50`) | Simple, random page access | Slow on deep pages (`OFFSET` scans), unstable under inserts (duplicates/skips) |
| **Cursor/keyset** | Fast, stable, scales | No random access; cursor must encode sort key (opaque base64) |

Default to cursor pagination for feeds/lists; cap `limit` (e.g. max 100).

## Idempotency and safe retries
Networks retry. For `POST /expenses` accept an **`Idempotency-Key`** header: store (key → response) for 24 h; a retry with the same key returns the original result rather than creating a duplicate.

```
POST /expenses
Idempotency-Key: 4c2e8f0a-...
```
Optimistic concurrency: return `ETag: "v7"` on GET; require `If-Match: "v7"` on PATCH/PUT; mismatch → `412 Precondition Failed` (or version field → 409). Prevents lost updates between two tabs.

## Versioning and evolution
- Prefer **additive, backward-compatible** changes (new optional fields) so you rarely need versions.
- When breaking: URL (`/v2/expenses`) is the most explicit and debuggable; header/media-type versioning is purer but harder to use. Pick one, document deprecation (`Deprecation`/`Sunset` headers).
- Clients must ignore unknown fields (tolerant reader).

## Cross-cutting concerns
- **Auth**: `Authorization: Bearer <JWT>` ([02](02-authentication-jwt-oauth.md)).
- **CORS** for the SPA origin; preflight caching (`Access-Control-Max-Age`).
- **Rate limiting/throttling** at API Gateway/WAF; 429 + `Retry-After`.
- **Caching**: `Cache-Control`, `ETag`/`If-None-Match` → `304`. Never cache per-user data in shared caches without `Vary`/`private`.
- **Compression**, **timeouts**, **request size limits**.
- **Observability**: request ID (`X-Request-Id`) propagated and logged; structured logs; metrics per route.
- **Documentation**: OpenAPI 3 spec as the contract (FastAPI generates it; Go via swaggo/oapi-codegen). Generate TypeScript clients/types from it ([../typescript/05](../typescript/05-typescript-with-react-and-node.md)).
- **Security**: validate everything, parameterized queries, HTTPS only, ownership checks (BOLA), avoid verbose errors, OWASP API Top 10.

## Worked example: the expense API contract (OpenAPI excerpt)

```yaml
openapi: 3.0.3
info: { title: Expense Tracker API, version: 1.0.0 }
paths:
  /expenses:
    get:
      summary: List expenses
      parameters:
        - { name: from, in: query, schema: { type: string, format: date } }
        - { name: to, in: query, schema: { type: string, format: date } }
        - { name: category, in: query, schema: { type: string } }
        - { name: limit, in: query, schema: { type: integer, minimum: 1, maximum: 100, default: 50 } }
        - { name: cursor, in: query, schema: { type: string } }
      responses:
        "200": { description: OK, content: { application/json: { schema: { $ref: "#/components/schemas/ExpensePage" } } } }
        "401": { $ref: "#/components/responses/Unauthorized" }
    post:
      summary: Create expense
      parameters:
        - { name: Idempotency-Key, in: header, schema: { type: string } }
      requestBody:
        required: true
        content: { application/json: { schema: { $ref: "#/components/schemas/CreateExpense" } } }
      responses:
        "201": { description: Created, headers: { Location: { schema: { type: string } } } }
        "422": { description: Validation error, content: { application/problem+json: {} } }
components:
  schemas:
    CreateExpense:
      type: object
      required: [amountCents, currency, category, date]
      properties:
        amountCents: { type: integer, minimum: 1 }
        currency: { type: string, minLength: 3, maxLength: 3 }
        category: { type: string }
        description: { type: string, maxLength: 200 }
        date: { type: string, format: date }
```

## REST vs GraphQL vs gRPC (comparison question)

| | REST | GraphQL | gRPC |
|---|---|---|---|
| Strength | Simple, cacheable, ubiquitous | Client chooses fields; one round trip for graphs | Fast binary, streaming, strong contracts |
| Weakness | Over/under-fetching | Caching, N+1, complexity, authz per field | Browser support needs gRPC-Web, less human-friendly |
| Use | Public/CRUD APIs, this project | Many varied clients / complex UI data needs | Internal service-to-service |

## Exercise
Write the OpenAPI for all expense endpoints (including errors), then implement **idempotent POST** and **cursor pagination** in FastAPI ([../fastapi/](../fastapi/)) or Gin ([../gin/](../gin/)).

## Interview Q&A
- **PUT vs PATCH?** Full replace (idempotent) vs partial change.
- **401 vs 403?** Who are you? vs you may not.
- **How do you make POST safe to retry?** Idempotency keys.
- **How do you paginate a large, changing dataset?** Keyset/cursor.
- **How do you prevent lost updates?** ETag + If-Match or version column.
- **How do you version an API?** Additive changes first; explicit `/v2` when breaking; deprecate with `Sunset`.
- **What is HATEOAS?** Hypermedia links in responses to drive client navigation; rarely implemented fully in practice.
