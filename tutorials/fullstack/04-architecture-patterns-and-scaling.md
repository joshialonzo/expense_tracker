# Full-Stack 04 — Application Architecture, Patterns and Scaling

## Why it matters
Full-stack interviews test whether you can structure code and systems so they stay maintainable and scale. Use the expense tracker as the concrete case.

> **This section is the short version.** The full treatment (dependency rule, SOLID applied, ports/adapters, composition root, contract tests, enforcement in CI) is [08-solid-and-clean-architecture.md](08-solid-and-clean-architecture.md), with concrete implementations in [../fastapi/07](../fastapi/07-clean-architecture.md), [../gin/07](../gin/07-clean-architecture.md), [../react/08](../react/08-clean-architecture-frontend.md) and [../angular/06](../angular/06-clean-architecture-frontend.md). Naming used there: `domain → application (use cases + ports) → adapters/interface → container (composition root)`.

## Layered backend (works in FastAPI, Gin, Express)

```
HTTP layer (routers/handlers)   → parse/validate input, auth, map errors → HTTP
Service layer (use cases)       → business rules, transactions, orchestration
Repository layer                → persistence (SQL/DynamoDB), returns domain objects
Domain models                   → entities & value objects (Expense, Money)
Infrastructure / adapters       → S3, Bedrock, email, MCP, clock
```
- **Dependency direction**: handlers → services → repositories (interfaces). Services depend on *abstractions* so they can be tested with fakes (**ports & adapters / hexagonal**).
- **DTOs vs entities**: API schemas ≠ DB models ≠ domain models (until you've proven they're the same, which they rarely stay).
- **Dependency injection**: FastAPI `Depends`, Go constructor injection, Angular DI, React context.

### Example (Python-ish) — service testable without a DB

```python
class ExpenseRepo(Protocol):
    def add(self, e: Expense) -> Expense: ...
    def list(self, user_id: str, start: date, end: date) -> list[Expense]: ...

class ExpenseService:
    def __init__(self, repo: ExpenseRepo, clock: Callable[[], datetime]): ...
    def create(self, user_id: str, data: CreateExpense) -> Expense:
        if data.date > self.clock().date():
            raise DomainError("future_date", "date cannot be in the future")
        return self.repo.add(Expense(user_id=user_id, **data.model_dump()))
```

## 12-factor app (shorthand answers)
Config in env vars/Secrets Manager; one codebase, many deploys; stateless processes (state in DB/cache); logs to stdout (CloudWatch collects); dependencies explicit; dev/prod parity (Docker); disposable processes with graceful shutdown (handle SIGTERM, drain requests, important for ECS rolling deploys).

## Monolith vs microservices
Start with a **modular monolith**: clear module boundaries, single deploy. Split into services when there is an organizational or scaling reason (independent teams/deploy cadence, very different scaling or runtime needs, e.g. the MCP/AI service). Microservices cost: network failures, distributed transactions, observability, versioning, operational overhead. The best answer shows judgment rather than dogma.

## Sync vs async communication
- **Request/response (HTTP/gRPC)**: user is waiting, need an answer now.
- **Async messaging**: decouple, absorb spikes, retry. SQS (queue), SNS (fan-out), EventBridge (routing), Kinesis/Kafka (streams).

Example: `POST /receipts` → store in S3 → S3 event → SQS → Lambda OCR (Textract/Bedrock) → update expense. API returns `202 Accepted` quickly; the UI polls or gets a push (WebSocket/SSE).

Patterns: **idempotent consumers** (deduplicate by message id), **dead-letter queues**, **retry with exponential backoff + jitter**, **outbox pattern** (write DB row + event in one transaction, relay later → avoids dual-write inconsistencies), **saga** for multi-step workflows with compensation.

## Scaling a web app: the ladder

1. Profile & fix the obvious (indexes, N+1, payload size).
2. **Stateless horizontal scaling**: more Lambda/ECS tasks behind a load balancer; sessions/state out of process.
3. **Cache**: CDN for static + cacheable GETs, Redis for hot data (monthly summaries).
4. **Database**: connection pooling → read replicas (route reports to replicas) → partitioning → sharding as a last resort.
5. **Async the slow stuff**: queues/workers for exports, AI summaries, emails.
6. **Rate limit & backpressure**: protect downstreams (Bedrock quotas!).
7. **Multi-AZ / multi-region** for availability.

Capacity back-of-envelope: 10k users × 5 expenses/day = 50k writes/day ≈ 0.6 writes/s average (peaks ×10 ≈ 6/s). A single Postgres handles that trivially → say so; don't over-engineer. Reads ≈ 10× writes.

## Reliability patterns
- **Timeouts everywhere**, **retries with backoff** (only idempotent ops), **circuit breaker** for flaky dependencies, **bulkheads** (separate pools), **graceful degradation** (dashboard still loads if AI insights fail), **health checks** (liveness vs readiness), **feature flags** for safe rollout.
- Define SLOs; handle failure modes explicitly (what if Bedrock is down? S3 upload fails after the expense is saved?).

## Observability
Structured logs (JSON) with request IDs, metrics (RED: Rate, Errors, Duration), distributed traces (OpenTelemetry), dashboards and alerts, audit logs for sensitive actions. See [../aws/08](../aws/08-cicd-observability-and-cost.md).

## Frontend architecture in the full picture
- SPA ↔ API contract via OpenAPI; generate types.
- Client caching with TanStack Query; optimistic UI.
- Error handling policy: global 401 handler, toast for transient errors, error boundary for render failures, retry for network errors.
- Feature flags, analytics, and performance budgets.
- Monorepo layout for this project:

```
expense_tracker/
  web-react/      React + TS (Vite)         | web-angular/  Angular + TS
  api-fastapi/    Python API                | api-gin/ (Go API), same OpenAPI contract
  mcp-server/     MCP server (REST adapter)  | assistant/  Bedrock agent loop over MCP tools
  infra/          CDK/SAM/Terraform
  tutorials/
  .github/workflows/
```
Duplicating front-ends/back-ends against a **shared OpenAPI contract** is exactly how to practice: any front-end must work with any back-end.

## Design patterns worth naming
Repository, Factory, Strategy (swap Bedrock/Azure provider behind an interface), Adapter (REST/MCP over the same service), Observer/pub-sub (events), Decorator/middleware (auth, logging), CQRS (separate read models for reports), Singleton (careful; prefer DI).

## SOLID in one line each
Single responsibility; Open/closed (extend without modifying); Liskov substitution; Interface segregation (small interfaces, idiomatic in Go); Dependency inversion (depend on abstractions).

## Code quality and reviews (JD: "write clean code", "participate in code reviews")
Small PRs, descriptive names, tests with changes, no dead code, consistent formatting (automated), review for correctness → security → design → readability → performance; give actionable, kind feedback; use ADRs (architecture decision records) to document significant decisions.

## Exercise
Refactor a handler-heavy endpoint into handler → service → repo with a fake repo and unit tests. Then add an async "export CSV" job using SQS + worker and return `202` with a polling endpoint.

## Interview Q&A
- **How do you structure a backend?** Layers with inward dependencies; DI; thin handlers.
- **Monolith vs microservices?** Start modular monolith; split with a reason.
- **How do you handle a slow third-party (Bedrock) call?** Timeout, retry w/ backoff, circuit breaker, async job, streaming UX, fallback.
- **What is the outbox pattern?** Atomically persist the event with the data, publish later.
- **Eventual consistency in the UI?** Optimistic updates, polling/refetch, clear status states.
- **How do you do zero-downtime deployments?** Backward-compatible changes, rolling/blue-green, health checks, graceful shutdown, expand/contract migrations.
