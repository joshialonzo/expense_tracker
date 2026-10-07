# Full-Stack 06 — System Design Walkthrough: The Expense Tracker (and AI features)

## Why it matters
"Design X" is the standard senior-ish full-stack question. Use a repeatable framework and this project as the worked example. Practice talking for 35–45 minutes.

## Framework (state it at the start)
1. **Clarify requirements**: functional, non-functional (scale, latency, availability, security, cost), out of scope.
2. **Estimate** load and storage.
3. **API** and **data model**.
4. **High-level architecture** (boxes and arrows).
5. **Deep dive** on 1–2 hard parts (interviewer's pick).
6. **Bottlenecks, failure modes, trade-offs, evolution.**

## 1. Requirements

Functional: sign up/in; CRUD expenses with categories; receipt upload; monthly reports; budgets and alerts; export; AI insights ("why was October so high?").
Non-functional: 100k users, p95 API < 300 ms, 99.9% availability, data durability, secure (PII/finance), GDPR delete/export, low cost at low scale.
Out of scope (say it): bank-account syncing (Plaid), multi-currency FX history, team accounts.

## 2. Estimation

- 100k users × 5 expenses/day = 500k writes/day ≈ 6/s average, ~60/s peak.
- Reads 10× → ~600/s peak (dashboard loads).
- Row ≈ 300 B → 500k/day × 365 × 300 B ≈ 55 GB/year. Fits comfortably on one Postgres (plus indexes ~2×).
- Receipts: 20% have a 500 KB image → 50 GB/year in S3 (cheap).
- AI: 10% of users/day × 1 insight × ~3k tokens = ~1M tokens/day, so cost is controlled by model choice and caching.

Conclusion: no need for sharding or a NoSQL-first design. Relational + cache + queue is plenty.

## 3. API and data model
See [01](01-rest-api-design.md) for the API and [../aws/05](../aws/05-rds-and-dynamodb.md) / [03](03-sql-and-nosql-modeling.md) for the schema. Additional tables: `budgets`, `receipts(expense_id, s3_key, status)`, `insights(user_id, month, content, created_at, model_id, prompt_version)`.

## 4. High-level architecture

```
                         ┌───────────────┐
 Browser ──HTTPS──► Route53 ─► CloudFront ──► S3 (React/Angular SPA)
    │                            │ WAF
    │ /api/*                     ▼
    └──────────────────► API Gateway (HTTP API, Cognito JWT authorizer, throttling)
                                 │
                 ┌───────────────┼──────────────────────┐
                 ▼               ▼                      ▼
            Lambda/ECS API   ECS Assistant API     Lambda presign
          (FastAPI or Gin)   (Bedrock + MCP)       (S3 uploads)
                 │               │  └─► MCP servers (expense tools, user-scoped)
                 ▼               ▼
   RDS Postgres (Multi-AZ) ◄─ RDS Proxy      Bedrock (Converse + Guardrails)
        │  └─ read replica (reports)
        ▼
   ElastiCache Redis (summaries, rate limits)

 S3 receipts ─event─► SQS ─► Lambda OCR/extract (Textract/Bedrock) ─► update expense, notify
 EventBridge schedule ─► Lambda: monthly budget alerts ─► SES/SNS
 CloudWatch + X-Ray + alarms + Budgets;  CI/CD: GitHub Actions (OIDC) → ECR/ECS, S3/CloudFront
```

Decisions to justify:
- **HTTP API + JWT authorizer**: auth offloaded, cheap.
- **Lambda vs ECS** for the API: Lambda is great at this scale and idle-cost; move to ECS if steady traffic/cold starts/long connections hurt ([../aws/04](../aws/04-ecs-fargate-and-ecr.md)).
- **Postgres**: relational reports, constraints; RDS Proxy for Lambda connections.
- **Async receipts**: S3 presigned upload + event-driven processing keeps the API fast; status field + polling/WebSocket.
- **Cache**: monthly summary keyed `summary:{user}:{month}`, invalidated on expense write.

## 5. Deep dives (pick the one asked)

### A. Monthly report performance
Naively `GROUP BY` over a user's rows is fast with the `(user_id, spent_on)` index (hundreds–thousands of rows per user per month). If needed: materialized summary table `monthly_totals(user_id, month, category, total_cents)` updated in the same transaction as expense writes (or via trigger/outbox), making reads O(categories). Trade-off: write amplification and correctness on edits/deletes (apply delta).

### B. AI insights feature
- Flow: `POST /insights {month}` → gather aggregates (not raw rows when possible: smaller, cheaper, less sensitive) → prompt → Bedrock → validate JSON → store → return. Cache by `(user, month, data-hash, prompt_version)`.
- Slow? Return `202`, process async (SQS/Lambda), push result via polling or WebSocket; or stream tokens over SSE.
- Quality: golden sets, schema validation, "say unknown if not in data", Guardrails for PII.
- Cost: smaller model for classification/summaries, token caps, per-user daily quota (Redis counter), batch off-peak.
- Safety: expense descriptions are untrusted user text and could contain prompt injection, so delimit them, never let model output trigger writes without validation/confirmation.
- Agentic version: MCP tools scoped to the caller ([../mcp/](../mcp/)).

### C. Auth & multi-tenancy
Cognito user pool; `sub` is the tenant key; every table has `user_id`; row-level checks in repository layer (or Postgres RLS: `CREATE POLICY … USING (user_id = current_setting('app.user_id')::uuid)`); tests proving isolation.

### D. Offline-first / mobile (stretch)
Client-side queue with idempotency keys, sync endpoint with `updated_since`, conflict resolution (last-write-wins vs versioned merge).

### E. Duplicate/idempotent writes and retries
`Idempotency-Key` table `(user_id, key) → response`, unique constraint, TTL cleanup.

## 6. Failure modes (be explicit)

| Failure | Behavior |
|---|---|
| DB primary fails | Multi-AZ failover (~60–120 s); app retries with backoff; reads from replica degrade gracefully |
| Bedrock throttled/down | Circuit breaker; insights show "temporarily unavailable"; core expenses unaffected |
| S3 upload ok but PATCH fails | Orphan object → lifecycle rule cleans unreferenced keys; client retries PATCH |
| Lambda concurrency spike | Throttle at gateway, queue non-urgent work, RDS Proxy protects DB |
| Bad deploy | Automatic rollback on alarm; backward-compatible schema changes |
| Region outage | RTO/RPO decision: backups + IaC redeploy (hours) vs active-passive (minutes) vs active-active (expensive); justify by business need |

## 7. Evolution
Add search (OpenSearch), recurring expenses scheduler, shared budgets (many-to-many), bank sync (webhooks + queue), analytics warehouse (S3 + Athena/Redshift) fed by CDC, mobile app (same API), multi-region.

## 8. Cost sketch (order of magnitude, verify with the AWS Pricing Calculator)
Small scale: S3+CloudFront pennies, Lambda/HTTP API ~free tier, RDS `db.t4g.small` Multi-AZ is the dominant fixed cost, NAT Gateway often next (use VPC endpoints), Bedrock proportional to tokens. At 100k users: add Redis, read replica, ECS/Fargate for steady traffic, reserved/savings plans.

## Practice prompts (do each aloud, 35 min)
1. Design the expense tracker (above).
2. Design a URL shortener / rate limiter (classic).
3. Design a notification system (email/push for budget alerts).
4. Design a chat assistant over private user data with tool use (MCP + Bedrock).
5. Design a CSV bank-statement import with dedupe and categorization (async pipeline).

## Evaluation rubric (self-score)
- [ ] Asked clarifying questions and stated assumptions
- [ ] Did numbers, not hand-waving
- [ ] Clear API + data model
- [ ] Justified each AWS service choice and named alternatives
- [ ] Covered security, observability, failure, cost
- [ ] Identified bottlenecks and a plan to evolve
