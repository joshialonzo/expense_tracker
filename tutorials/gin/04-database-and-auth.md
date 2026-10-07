# Go 04 — PostgreSQL (pgx), Migrations and JWT Auth in Gin

## Database access options
- `database/sql` + driver: stdlib, portable.
- **`pgx`** (`github.com/jackc/pgx/v5`, with `pgxpool`): native Postgres driver, best features/perf.
- **`sqlc`**: write SQL, generate type-safe Go (popular, no runtime magic).
- GORM/ent: ORMs. For interviews: know raw SQL + one of sqlc/pgx; mention ORM trade-offs (magic, N+1, harder to optimize).

```bash
go get github.com/jackc/pgx/v5
go install github.com/golang-migrate/migrate/v4/cmd/migrate@latest   # needs -tags 'postgres' build, or use brew install golang-migrate
```

## Migrations (golang-migrate)

```bash
migrate create -ext sql -dir migrations -seq create_expenses
```
`migrations/000001_create_expenses.up.sql` (same schema as [../aws/05](../aws/05-rds-and-dynamodb.md)) and `.down.sql` that reverses it.

```bash
migrate -path migrations -database "$DATABASE_URL" up
```
Run as a CI/CD step or ECS one-off task before the service rolls.

## Pool

```go
func mustPool(url string) *pgxpool.Pool {
	cfg, err := pgxpool.ParseConfig(url)
	if err != nil { log.Fatal(err) }
	cfg.MaxConns = 10
	cfg.MaxConnLifetime = 30 * time.Minute
	cfg.HealthCheckPeriod = time.Minute
	pool, err := pgxpool.NewWithConfig(context.Background(), cfg)
	if err != nil { log.Fatal(err) }
	return pool
}
```

> **Clean-architecture note:** the Postgres repository is an adapter (`internal/adapters/pg`) satisfying the same `app.ExpenseRepository` as the DynamoDB and in-memory ones; run the shared contract suite ([07](07-clean-architecture.md)) against it.

## Repository (parameterized SQL, ownership in every query)

```go
type PgRepo struct{ db *pgxpool.Pool }
func NewPgRepo(db *pgxpool.Pool) *PgRepo { return &PgRepo{db} }

func (r *PgRepo) Create(ctx context.Context, userID string, in CreateInput) (Expense, error) {
	const q = `
		INSERT INTO expenses (user_id, amount_cents, currency, category, description, spent_on)
		VALUES ($1, $2, COALESCE(NULLIF($3,''),'USD'), $4, $5, $6)
		RETURNING id, amount_cents, currency, category, description, to_char(spent_on,'YYYY-MM-DD'), created_at`
	var e Expense
	err := r.db.QueryRow(ctx, q, userID, in.AmountCents, in.Currency, in.Category, in.Description, in.SpentOn).
		Scan(&e.ID, &e.AmountCents, &e.Currency, &e.Category, &e.Description, &e.SpentOn, &e.CreatedAt)
	return e, err
}

func (r *PgRepo) Get(ctx context.Context, userID, id string) (Expense, error) {
	const q = `SELECT id, amount_cents, currency, category, description, to_char(spent_on,'YYYY-MM-DD'), created_at
	           FROM expenses WHERE id = $1 AND user_id = $2`
	var e Expense
	err := r.db.QueryRow(ctx, q, id, userID).Scan(&e.ID, &e.AmountCents, &e.Currency, &e.Category, &e.Description, &e.SpentOn, &e.CreatedAt)
	if errors.Is(err, pgx.ErrNoRows) { return Expense{}, ErrNotFound }
	return e, err
}

func (r *PgRepo) List(ctx context.Context, userID string, f ListFilter) ([]Expense, string, error) {
	const q = `
		SELECT id, amount_cents, currency, category, description, spent_on, created_at
		FROM expenses
		WHERE user_id = $1
		  AND ($2::date IS NULL OR spent_on >= $2)
		  AND ($3::date IS NULL OR spent_on <= $3)
		  AND ($4::text IS NULL OR category = $4)
		  AND ($5::date IS NULL OR (spent_on, id) < ($5, $6::uuid))
		ORDER BY spent_on DESC, id DESC
		LIMIT $7`
	rows, err := r.db.Query(ctx, q, userID, f.From, f.To, f.Category, f.CursorDate, f.CursorID, f.Limit+1)
	if err != nil { return nil, "", err }
	defer rows.Close()
	// pgx.CollectRows(rows, pgx.RowToStructByName[...]) simplifies scanning
	...
}
```
Never build SQL with string concatenation of user input; use `$n` parameters. `sqlc` would generate the above from a `.sql` file with compile-time checking.

### Transactions

```go
func (r *PgRepo) Transfer(ctx context.Context, ...) error {
	tx, err := r.db.Begin(ctx)
	if err != nil { return err }
	defer tx.Rollback(ctx) // no-op after Commit
	if _, err := tx.Exec(ctx, `UPDATE ...`); err != nil { return err }
	if _, err := tx.Exec(ctx, `UPDATE ...`); err != nil { return err }
	return tx.Commit(ctx)
}
```

## Verifying Cognito JWTs

```go
go get github.com/golang-jwt/jwt/v5 github.com/MicahParks/keyfunc/v3
```

```go
// internal/auth/jwt.go
package auth

type Verifier struct {
	keyfunc  keyfunc.Keyfunc
	issuer   string
	clientID string
}

func NewCognitoVerifier(region, poolID, clientID string) (*Verifier, error) {
	issuer := fmt.Sprintf("https://cognito-idp.%s.amazonaws.com/%s", region, poolID)
	k, err := keyfunc.NewDefault([]string{issuer + "/.well-known/jwks.json"}) // background refresh + caching
	if err != nil { return nil, err }
	return &Verifier{keyfunc: k, issuer: issuer, clientID: clientID}, nil
}

type Claims struct {
	jwt.RegisteredClaims
	TokenUse string   `json:"token_use"`
	ClientID string   `json:"client_id"`
	Groups   []string `json:"cognito:groups"`
}

func (v *Verifier) Verify(raw string) (*Claims, error) {
	var c Claims
	tok, err := jwt.ParseWithClaims(raw, &c, v.keyfunc.Keyfunc,
		jwt.WithValidMethods([]string{"RS256"}),   // pin the algorithm
		jwt.WithIssuer(v.issuer),
		jwt.WithExpirationRequired(),
		jwt.WithLeeway(5*time.Second))
	if err != nil || !tok.Valid { return nil, fmt.Errorf("invalid token: %w", err) }
	if c.TokenUse != "access" || c.ClientID != v.clientID { return nil, errors.New("wrong token") }
	return &c, nil
}

func Required(v *Verifier) gin.HandlerFunc {
	return func(c *gin.Context) {
		h := c.GetHeader("Authorization")
		raw, ok := strings.CutPrefix(h, "Bearer ")
		if !ok {
			c.Header("WWW-Authenticate", "Bearer")
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"status": 401, "detail": "missing token"})
			return
		}
		claims, err := v.Verify(raw)
		if err != nil {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"status": 401, "detail": "invalid token"})
			return
		}
		c.Set("userID", claims.Subject)
		c.Set("groups", claims.Groups)
		c.Next()
	}
}

func RequireGroup(g string) gin.HandlerFunc {
	return func(c *gin.Context) {
		groups, _ := c.Get("groups")
		if gs, ok := groups.([]string); ok && slices.Contains(gs, g) { c.Next(); return }
		c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"status": 403, "detail": "forbidden"})
	}
}
```
(Library APIs evolve—check the `keyfunc` and `golang-jwt` docs for the version you install.)

## Integration tests with a real Postgres (testcontainers-go)

```go
func TestRepo_ListIsOwnerScoped(t *testing.T) {
	ctx := context.Background()
	pg, err := postgres.Run(ctx, "postgres:16", postgres.WithDatabase("test"), postgres.WithUsername("u"), postgres.WithPassword("p"),
		testcontainers.WithWaitStrategy(wait.ForLog("ready to accept connections").WithOccurrence(2)))
	require.NoError(t, err)
	t.Cleanup(func() { _ = pg.Terminate(ctx) })

	url, _ := pg.ConnectionString(ctx, "sslmode=disable")
	pool := mustPool(url)
	applyMigrations(t, url)

	repo := NewPgRepo(pool)
	_, _ = repo.Create(ctx, "alice", CreateInput{AmountCents: 100, Category: "food", SpentOn: "2026-10-01"})
	got, _, err := repo.List(ctx, "bob", ListFilter{Limit: 10})
	require.NoError(t, err)
	require.Empty(t, got)                    // bob can't see alice's data
}
```

## Exercise
Replace the in-memory repo with `PgRepo`, add keyset pagination, protect routes with the JWT middleware, and add tests: no token, expired token, wrong user (404), pagination walk with 120 rows.

## Interview Q&A
- **`database/sql` vs pgx vs ORM?** Portability vs features/perf vs productivity/magic; sqlc as a middle path.
- **How do you manage connections in Go?** `pgxpool` sized to instance count × max conns ≤ DB limit; RDS Proxy.
- **How do you stop SQL injection?** Parameter placeholders always.
- **How do you handle `NULL`?** Pointers (`*string`) or `pgtype`/`sql.Null*`.
- **Why pin `alg` in JWT parsing?** Prevent algorithm-confusion/`none` attacks.
