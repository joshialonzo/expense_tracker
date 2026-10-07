# Full-Stack 03 — SQL and NoSQL: Modeling, Querying, Transactions

## Why it matters
JD: "Solid understanding of database systems, SQL, and NoSQL." Expect writing queries on a whiteboard, designing a schema, explaining indexes/transactions, and choosing SQL vs NoSQL. AWS specifics are in [../aws/05](../aws/05-rds-and-dynamodb.md).

## Relational modeling basics
- Tables, rows, primary keys, foreign keys, constraints (`NOT NULL`, `UNIQUE`, `CHECK`), **normalization** (avoid duplicating facts: 1NF atomic values, 2NF/3NF no partial/transitive dependencies). Denormalize deliberately for read performance.
- Relationships: one-to-many (FK on the many side), many-to-many (join table).

Extend the tracker with budgets and tags:

```sql
CREATE TABLE categories (
  id      SERIAL PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES users(id),
  name    TEXT NOT NULL,
  UNIQUE (user_id, name)
);

CREATE TABLE budgets (
  user_id      UUID NOT NULL REFERENCES users(id),
  category_id  INT  NOT NULL REFERENCES categories(id),
  month        DATE NOT NULL,                 -- first day of the month
  limit_cents  BIGINT NOT NULL CHECK (limit_cents >= 0),
  PRIMARY KEY (user_id, category_id, month)
);

CREATE TABLE tags (id SERIAL PRIMARY KEY, name TEXT UNIQUE NOT NULL);
CREATE TABLE expense_tags (
  expense_id UUID REFERENCES expenses(id) ON DELETE CASCADE,
  tag_id     INT  REFERENCES tags(id),
  PRIMARY KEY (expense_id, tag_id)
);
```

## SQL you must write fluently

```sql
-- Filter, sort, limit
SELECT * FROM expenses WHERE user_id = $1 AND spent_on >= date_trunc('month', now()) ORDER BY spent_on DESC LIMIT 20;

-- Aggregate + HAVING
SELECT category, SUM(amount_cents) AS total
FROM expenses WHERE user_id = $1
GROUP BY category HAVING SUM(amount_cents) > 10000 ORDER BY total DESC;

-- JOINs
SELECT e.id, e.amount_cents, b.limit_cents
FROM expenses e
LEFT JOIN categories c ON c.user_id = e.user_id AND c.name = e.category
LEFT JOIN budgets b    ON b.category_id = c.id AND b.month = date_trunc('month', e.spent_on)
WHERE e.user_id = $1;

-- CTE + window function: running total and rank within each category
WITH monthly AS (
  SELECT category, date_trunc('month', spent_on) AS month, SUM(amount_cents) AS total
  FROM expenses WHERE user_id = $1 GROUP BY 1, 2
)
SELECT category, month, total,
       SUM(total) OVER (PARTITION BY category ORDER BY month) AS running_total,
       RANK()     OVER (PARTITION BY month ORDER BY total DESC) AS rank_in_month,
       total - LAG(total) OVER (PARTITION BY category ORDER BY month) AS change_vs_prev
FROM monthly;

-- Top N per group: biggest 3 expenses per category
SELECT * FROM (
  SELECT e.*, ROW_NUMBER() OVER (PARTITION BY category ORDER BY amount_cents DESC) AS rn
  FROM expenses e WHERE user_id = $1
) t WHERE rn <= 3;

-- Find duplicates
SELECT user_id, spent_on, amount_cents, description, COUNT(*)
FROM expenses GROUP BY 1,2,3,4 HAVING COUNT(*) > 1;

-- Upsert
INSERT INTO budgets (user_id, category_id, month, limit_cents) VALUES ($1,$2,$3,$4)
ON CONFLICT (user_id, category_id, month) DO UPDATE SET limit_cents = EXCLUDED.limit_cents;
```

Join types: INNER (matches only), LEFT (all left + matches), RIGHT, FULL, CROSS. `WHERE` filters rows before grouping, `HAVING` filters groups. `NULL` semantics: `= NULL` is never true; use `IS NULL`; `COUNT(col)` skips nulls, `COUNT(*)` doesn't.

## Indexes
- B-tree is default: equality + range + sort. Multi-column index is useful left-to-right (leftmost prefix rule): `(user_id, spent_on)` serves `WHERE user_id=?` and `WHERE user_id=? AND spent_on BETWEEN…`, not `WHERE spent_on…` alone.
- Covering indexes (`INCLUDE`), partial indexes (`WHERE deleted_at IS NULL`), GIN for JSONB/full-text, expression indexes.
- Cost: slower writes, more storage. Index for real queries, verify with `EXPLAIN (ANALYZE, BUFFERS)`: look for `Seq Scan` on big tables, row estimate vs actual, `Index Only Scan`.
- Why an index might not be used: function on the column (`WHERE date(spent_on)=…`), type mismatch, low selectivity, stale stats (`ANALYZE`), leading wildcard `LIKE '%x'`.

## Transactions and isolation (ACID)
**A**tomicity, **C**onsistency, **I**solation, **D**urability.

```sql
BEGIN;
  UPDATE accounts SET balance_cents = balance_cents - 5000 WHERE id = 1;
  UPDATE accounts SET balance_cents = balance_cents + 5000 WHERE id = 2;
COMMIT;
```
Isolation levels (anomalies prevented): Read Committed (default in Postgres; no dirty reads) → Repeatable Read (no non-repeatable reads; snapshot) → Serializable (no phantoms/write skew). Higher = more conflicts/retries. Concurrency tools: `SELECT … FOR UPDATE` (pessimistic row lock), **optimistic locking** with a `version` column:

```sql
UPDATE expenses SET amount_cents = $1, version = version + 1 WHERE id = $2 AND version = $3;  -- 0 rows → conflict (409)
```
Deadlocks: acquire locks in a consistent order; keep transactions short.

## N+1 queries
Fetching a list then querying per row. Fix with a JOIN/`IN (...)`, eager loading (`selectinload` in SQLAlchemy), or DataLoader. Detect with query logging/APM.

## Migrations
Versioned, forward-only migrations in git (Alembic for SQLAlchemy, golang-migrate/goose for Go, Flyway/Liquibase). Run in CI/CD before the app rollout; **expand → migrate → contract** for zero downtime; never edit an applied migration.

## NoSQL landscape

| Type | Example | Best for |
|---|---|---|
| Key-value | DynamoDB, Redis | Lookups by key, caching, sessions |
| Document | DynamoDB (items), MongoDB | Aggregates read as a unit, flexible schema |
| Wide-column | Cassandra | Massive write throughput, time series |
| Graph | Neptune, Neo4j | Relationship traversal |
| Search | OpenSearch | Full-text, faceting |
| Time series | Timestream | Metrics |

NoSQL modeling flips relational thinking: **start from access patterns**, embed related data read together, duplicate to avoid joins, accept weaker ad-hoc queries.

### Document modeling example (embed vs reference)
Embed what's read together and bounded (an expense's `splits[]`); reference when unbounded or independently updated (a user's expenses → separate items keyed by user).

### CAP & consistency
During a partition you choose **C**onsistency or **A**vailability. Many NoSQL stores default to eventual consistency (BASE: basically available, soft state, eventually consistent) and offer tunable/strong reads. **PACELC** adds the latency/consistency tradeoff when there is no partition.

## SQL vs NoSQL decision

| Choose SQL when… | Choose NoSQL when… |
|---|---|
| Relationships, constraints, ad-hoc queries/reports, transactions across entities | Known access patterns at huge scale, flexible/evolving schema, single-digit-ms key lookups, spiky serverless |
| Team knows SQL; the data is small to large (Postgres scales far) | Need horizontal write scaling/automatic partitioning |

"It's rarely either/or" → **polyglot persistence**: Postgres system of record + Redis cache + OpenSearch for search + S3 for blobs.

## Caching (Redis/ElastiCache)
Cache-aside: read cache → on miss read DB → populate with TTL; invalidate or expire on write. Problems: stale data, **cache stampede** (use locks/jitter/early refresh), cache penetration. Cache only what's read often and expensive (monthly summary).

## ORMs and drivers
- Python: SQLAlchemy 2.0 + Alembic ([../fastapi/03](../fastapi/03-database-with-sqlalchemy.md)). Go: `pgx` + `sqlc` or GORM ([../gin/04](../gin/04-database-and-auth.md)). Node: Prisma/Drizzle/Knex.
- Know when to drop to raw SQL (reports, bulk ops). Always use bind parameters.
- Connection pooling (pgbouncer/RDS Proxy); pool size × instances ≤ DB max connections.

## Exercise
1. Load 1M expenses with `generate_series`; run the monthly summary with and without `(user_id, spent_on)`; record `EXPLAIN ANALYZE` times.
2. Write the window-function query for "month-over-month change per category".
3. Model "shared expenses with splits" in Postgres and in a document store; compare.

## Interview Q&A
- **Clustered vs non-clustered index?** Clustered defines physical row order (InnoDB PK); Postgres heap tables use secondary indexes and `CLUSTER` is one-off.
- **Why avoid `SELECT *`?** Extra I/O, breaks covering indexes, brittle to schema changes.
- **Explain isolation levels with an example.** Write skew example: two doctors on call both go off-call at the same time under snapshot isolation.
- **How do you scale a relational DB?** Index/query tuning → caching → read replicas → partitioning/sharding → CQRS/separate analytics store.
- **What is a covering index?** Contains all columns a query needs → index-only scan.
- **How do you do soft delete?** `deleted_at` + partial indexes + default filters; consider audit/retention (GDPR erasure).
