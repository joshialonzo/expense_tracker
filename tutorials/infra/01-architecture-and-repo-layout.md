# Infra 01 — Architecture, Variants and Repo Layout

## Goal
Deploy three interchangeable versions of the expense tracker from **one infrastructure codebase**, each with Bedrock and MCP:

| Variant id | Front-end | Back-end | Data |
|---|---|---|---|
| `react-fastapi` | React + TypeScript | FastAPI (Python) | DynamoDB |
| `angular-fastapi` | Angular + TypeScript | FastAPI (Python) | DynamoDB |
| `react-gin` | React + TypeScript | Gin (Go) | DynamoDB |

Each variant is a **separate, isolated set of stacks** (own Cognito pool, table, API, CloudFront), so you can run all three side by side and compare them. All are serverless and pay-per-use, so idle cost is close to zero.

Tool choice: **AWS CDK in TypeScript** (matches the TypeScript focus, real code instead of YAML, and you can discuss CDK vs Terraform vs SAM from [../aws/08](../aws/08-cicd-observability-and-cost.md)).

## Architecture

```
Browser
  │ https://dxxxx.cloudfront.net
  ▼
CloudFront ─────────────── default behavior ──► S3 (private, OAC)   React or Angular build + config.json
  │   (viewer-request function: SPA rewrite → /index.html)
  │
  └── /api/*  (function strips "/api")
        ▼
   API Gateway HTTP API  ── Cognito JWT authorizer (issuer + client id) ──
        │
        ├─ ANY /expenses, /expenses/*, /reports/*, /receipts/*  ─► Lambda "api"        (FastAPI | Gin container)
        ├─ POST /assistant                                       ─► Lambda "assistant"  (FastAPI + MCP client + Bedrock)
        └─ POST|GET|DELETE /mcp                                  ─► Lambda "mcp"        (FastMCP, stateless Streamable HTTP)

   "api" ───► DynamoDB single table + S3 receipts bucket (presigned URLs)
   "assistant" ──► Bedrock Converse  ──► tools via MCP ──► /mcp ──► /expenses … (same API, caller's JWT)
   "mcp" ───────► calls the REST API with the caller's bearer token (no direct DB access)

Cognito (user pool + hosted UI, Authorization Code + PKCE)       CloudWatch logs/alarms, X-Ray, Budgets
```

### Why this shape (decisions you can defend in an interview)

1. **Two planes.**
   - *CRUD plane* (`api`): the only part that differs between FastAPI and Gin. It owns the data.
   - *AI plane* (`assistant` + `mcp`): identical for all variants. It talks only to the public REST contract, so **any back-end that honors the OpenAPI contract gets the AI features for free**. This is the "ports and adapters" argument from [../fullstack/04](../fullstack/04-architecture-patterns-and-scaling.md) made concrete.
2. **Lambda Web Adapter (LWA)** runs a normal HTTP server (uvicorn, Gin) inside Lambda, unchanged. No Mangum, no Go Lambda handler; the same Dockerfile pattern works for every service, and you can still run the app locally with `docker run`.
3. **CloudFront in front of everything** → one origin for browser (no CORS in production), static caching, TLS, and API Gateway isn't exposed to the SPA directly.
4. **JWT authorizer at the gateway** → unauthenticated traffic never invokes Lambda (cost and security). Services read the verified claims from the request context LWA forwards (`x-amzn-request-context`).
5. **MCP server is an adapter over REST with token passthrough.** The MCP tool calls the API *as the user*, so authorization and ownership checks stay in one place. (Token passthrough is acceptable here because the API is the token's intended audience. See the caveat in [../mcp/05](../mcp/05-security-and-deployment-on-aws.md).)
6. **DynamoDB single table** with on-demand billing: no VPC, no NAT, no connection pools; ideal for serverless and cheap at practice scale.
7. **Isolated variants** cost almost nothing extra and let you A/B the stacks (cold start, latency, bundle size).

8. **Clean architecture inside every service.** Each deployable (FastAPI, Gin, assistant, MCP server, React, Angular) has the same layers (`domain → application/ports → adapters → composition root`). The infrastructure choices above (DynamoDB, Bedrock, MCP, Lambda) are all *adapters*, so they are swappable: see [../fullstack/08](../fullstack/08-solid-and-clean-architecture.md), [../fastapi/07](../fastapi/07-clean-architecture.md), [../gin/07](../gin/07-clean-architecture.md), [../mcp/07](../mcp/07-mcp-and-assistant-as-adapters.md).

### Known trade-offs to mention
- API Gateway HTTP API has a **30 s** integration timeout, so the assistant is limited to a few tool turns. For long answers or token streaming use a Lambda Function URL with response streaming or ECS ([../aws/04](../aws/04-ecs-fargate-and-ecr.md)).
- The assistant → `/mcp` → `/expenses` chain makes three Lambda hops (more latency/cost) but proves the contract. A production build might have the assistant call tools in-process.
- Lambda cold starts on container images are slower than zip packages. Go is much faster than Python here; measure it ([08](08-operations-observability-cost-teardown.md)).

## DynamoDB key design (shared by FastAPI and Gin)

Access patterns first:

| # | Pattern | Index |
|---|---|---|
| 1 | Get/update/delete one expense | base table `GetItem(PK, SK)` |
| 2 | List my expenses in a date range, newest first | **GSI1** |
| 3 | Monthly summary by category | Query GSI1 for the month, aggregate in code |
| 4 | Category filter | filter expression on pattern 2 (fine at this scale) |

| Item | PK | SK | GSI1PK | GSI1SK |
|---|---|---|---|---|
| Expense | `USER#<sub>` | `EXP#<ulid>` | `USER#<sub>` | `DATE#2026-10-05#<ulid>` |
| Budget (future) | `USER#<sub>` | `BUDGET#2026-10#food` | | |
| Idempotency key | `USER#<sub>` | `IDEM#<key>` (with `ttl`) | | |

The ID is a **ULID** (time-sortable, URL-safe), so `GET /expenses/{id}` is a direct `GetItem` without knowing the date. This improves on the sketch in [../aws/05](../aws/05-rds-and-dynamodb.md), which put the date in the sort key and so needed the date to fetch one item. Every key starts with the Cognito `sub`, so a bug-free `PK = USER#<caller>` makes cross-user access structurally impossible.

## Repository layout

```
expense_tracker/
  web-react/            React + TS (Vite)                       → dist/
  web-angular/          Angular + TS                            → dist/web-angular/browser
  api-fastapi/          FastAPI service (DynamoDB repo)         Dockerfile (LWA)
  api-gin/              Gin service (DynamoDB repo)             Dockerfile (LWA)
  assistant/            FastAPI + MCP client + Bedrock          Dockerfile (LWA)
  mcp-server/           FastMCP server (REST adapter)           Dockerfile (LWA)
  infra/                CDK app (TypeScript)
    bin/app.ts
    lib/config.ts  data-stack.ts  auth-stack.ts  api-stack.ts  web-stack.ts
    scripts/deploy.sh  smoke-test.sh
  .github/workflows/deploy.yml
  tutorials/
```
(The earlier tutorials used `web/` and `mcp/`; use these names from now on.)

## Stack inventory per variant

| Stack | Contents | Depends on |
|---|---|---|
| `expense-<variant>-<stage>-data` | DynamoDB table (+GSI1), receipts bucket | none |
| `…-auth` | Cognito user pool, app client, hosted UI domain, `admin` group | web origin (context) |
| `…-api` | 3 Lambda container functions, HTTP API, JWT authorizer, IAM for DynamoDB/S3/Bedrock | data, auth |
| `…-web` | S3 bucket, CloudFront + functions, asset deployment, runtime `config.json` | api, auth |

Deploy order follows the dependency arrows, and CDK works it out from cross-stack references. The one circularity (Cognito callback URLs need the CloudFront domain, which exists only after the web stack) is solved with a two-pass deploy in [06](06-api-gateway-cloudfront-and-frontends.md).

## Tutorials in this series
1. This page
2. [Prerequisites, accounts and bootstrap](02-prerequisites-and-bootstrap.md)
3. [CDK project, data and auth stacks](03-cdk-project-data-and-auth.md)
4. [Backend containers: FastAPI and Gin on Lambda](04-backend-containers-fastapi-and-gin.md)
5. [AI plane: MCP server, assistant and Bedrock](05-ai-plane-mcp-and-bedrock.md)
6. [HTTP API, CloudFront and the two front-ends](06-api-gateway-cloudfront-and-frontends.md)
7. [CI/CD with GitHub Actions (matrix of variants)](07-cicd-github-actions.md)
8. [Operations, observability, cost and teardown](08-operations-observability-cost-teardown.md)
9. [Interview questions](09-interview-questions.md)

## Exercise
Draw this diagram from memory, then annotate every arrow with (a) the credential used (JWT, IAM role, presigned URL) and (b) what happens when that hop fails.
