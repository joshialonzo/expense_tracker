# AWS 05 — Databases: RDS (SQL) and DynamoDB (NoSQL)

## Why it matters
The JD asks for "SQL and NoSQL" and "RDS". The expense tracker is a perfect vehicle: relational data (reports, joins, constraints) versus key-access data (per-user lists).

## Choosing

| | **RDS / Aurora (PostgreSQL)** | **DynamoDB** |
|---|---|---|
| Model | Relational, joins, ACID, ad-hoc SQL | Key-value/document, access-pattern driven |
| Scaling | Vertical + read replicas | Virtually unlimited horizontally |
| Ops | Instances, patching windows, connections | Serverless, no connections |
| Queries | Anything (indexes help) | Only by key/GSI/LSI (design up front) |
| Best for | Reporting, relationships, flexible queries | Known access patterns, huge scale, spiky traffic |

For the tracker: **Postgres** as the system of record (summary reports are `GROUP BY`), and mention DynamoDB for the "list my recent expenses" hot path or session data. Say the trade-off aloud.

## RDS essentials
- **Multi-AZ**: synchronous standby in another AZ for **availability/failover** (not for reads). **Read replicas**: asynchronous copies for **read scaling** (eventual consistency).
- **Aurora**: AWS-engineered, storage auto-grows, faster failover, up to 15 replicas; **Aurora Serverless v2** scales capacity in fine increments.
- Backups: automated (point-in-time recovery) + manual snapshots. Encrypt with KMS at creation.
- Place in **private subnets**; security group allows 5432 only from the app's security group. Never `publicly accessible`.
- **Secrets Manager** stores credentials and can rotate them. App fetches at startup.
- **RDS Proxy** pools connections, essential with Lambda.
- Use IAM database authentication when you want tokens instead of passwords.

### Schema (PostgreSQL)

```sql
CREATE TABLE users (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email       TEXT NOT NULL UNIQUE,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE expenses (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  amount_cents  BIGINT NOT NULL CHECK (amount_cents > 0),
  currency      CHAR(3) NOT NULL DEFAULT 'USD',
  category      TEXT NOT NULL,
  description   TEXT,
  spent_on      DATE NOT NULL,
  receipt_key   TEXT,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Matches the main query: "my expenses in a date range"
CREATE INDEX idx_expenses_user_date ON expenses (user_id, spent_on DESC);
```

Design choices to articulate: money as integer cents (never floats), `TIMESTAMPTZ`, FK + cascade, composite index ordered by the query's equality column first then range/sort column.

### Report query

```sql
SELECT category, SUM(amount_cents) AS total_cents, COUNT(*) AS n
FROM expenses
WHERE user_id = $1 AND spent_on >= $2 AND spent_on < $3
GROUP BY category
ORDER BY total_cents DESC;
```
Use `EXPLAIN (ANALYZE, BUFFERS)` to prove the index is used. More in [../fullstack/03-sql-and-nosql-modeling.md](../fullstack/03-sql-and-nosql-modeling.md).

### Cursor pagination (not OFFSET)

```sql
SELECT * FROM expenses
WHERE user_id = $1 AND (spent_on, id) < ($2, $3)
ORDER BY spent_on DESC, id DESC
LIMIT 50;
```

## DynamoDB essentials
- **Partition key (PK)** determines the partition (hash); **sort key (SK)** orders items within it. A well-distributed PK avoids hot partitions.
- Reads: `GetItem`/`Query` (by PK + SK condition) are cheap; `Scan` reads everything, avoid it.
- **GSI** = alternate key schema, eventually consistent; **LSI** = same PK, different SK, must be defined at table creation.
- Capacity: **On-demand** (pay per request, spiky) vs **Provisioned** (+ auto scaling, cheaper at steady load).
- Consistency: eventual by default; **strongly consistent reads** optional on the base table (not GSIs).
- Transactions (`TransactWriteItems`), conditional writes (`attribute_not_exists`) for idempotency/optimistic locking, TTL for expiry, Streams for change data capture.

### Single-table design for the tracker

| PK | SK | Attributes |
|---|---|---|
| `USER#u1` | `PROFILE` | email |
| `USER#u1` | `EXP#2026-10-05#e9` | amountCents, category, … |
| `USER#u1` | `EXP#2026-10-06#e10` | … |

Access patterns:
1. List a user's expenses in a date range → `Query PK=USER#u1 AND SK BETWEEN EXP#2026-10-01 AND EXP#2026-10-31~`
2. Get a single expense → `GetItem`
3. By category → **GSI1**: `GSI1PK = USER#u1#CAT#food`, `GSI1SK = date`

```python
from boto3.dynamodb.conditions import Key

table.query(
    KeyConditionExpression=Key("PK").eq("USER#u1") & Key("SK").between("EXP#2026-10-01", "EXP#2026-10-31~"),
    ScanIndexForward=False,       # newest first
    Limit=50,
)
```
Pagination uses `LastEvaluatedKey` → `ExclusiveStartKey` (an opaque cursor you can base64 to the client).

Process: **list access patterns first**, then design keys. Aggregations (monthly totals) are awkward; precompute with Streams + Lambda or keep them in SQL.

## Exercise
1. Create the Postgres schema locally (Docker), load 100k rows, compare query plans with and without the index.
2. Model the same data in DynamoDB; implement patterns 1–3 and write down which are impossible without a GSI.

## Interview Q&A
- **Multi-AZ vs read replica?** HA/failover (sync, not readable) vs read scaling (async, readable).
- **How do you connect Lambda to RDS safely?** RDS Proxy; private subnets; credentials from Secrets Manager; limit concurrency.
- **Why is `Scan` bad?** Reads/bills every item; use keys/GSIs.
- **Hot partition?** Uneven key distribution throttles a partition; add write sharding / better keys.
- **ACID in DynamoDB?** Item-level atomic; multi-item via transactions (2× cost).
- **SQL injection prevention?** Parameterized queries / ORM binding — never string-format SQL.
- **Zero-downtime schema migration?** Expand → backfill → contract (add nullable column, dual-write, migrate, then drop old). Use Alembic/golang-migrate/Flyway in CI.
