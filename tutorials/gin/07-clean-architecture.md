# Go 07 — Clean Architecture and SOLID in Go (Gin)

Theory and rules: [../fullstack/08](../fullstack/08-solid-and-clean-architecture.md). This is the **reference structure for `api-gin/`**, the Go twin of [../fastapi/07](../fastapi/07-clean-architecture.md): same layers, same use cases, same contract tests, Go idioms.

> **Implemented and tested in [`api-gin/`](../../api-gin)** (module `expensetracker/api-gin`; the tutorial text uses the placeholder `github.com/YOUR_USER/...`). The latest Gin requires **Go 1.25**, so `go.mod` and the Dockerfile use it. The shared repository contract (`repotest.Run`) passes for the in-memory adapter and, against DynamoDB Local, for the real DynamoDB adapter.

## Go idioms that make this natural
- **Interfaces are satisfied implicitly** and should be **defined by the consumer** (the application layer), so adapters never import the interface: the dependency points inward automatically (DIP).
- **Small interfaces** (1–3 methods) are the language's native ISP.
- **`internal/`** blocks outside modules from importing your layers; **package boundaries** are the layer boundaries.
- Errors are values: domain errors are sentinels/typed errors; the HTTP adapter maps them with `errors.Is/As`.
- **Accept interfaces, return structs.**

## Layout

```
api-gin/
  cmd/api/main.go                       # composition root
  internal/
    domain/                             # pure Go: stdlib only
      money.go  expense.go  errors.go  summary.go
    app/                                # use cases + ports (consumer-defined interfaces). Imports domain only.
      ports.go
      create_expense.go  get_expense.go  list_expenses.go  delete_expense.go  monthly_summary.go
    adapters/
      ddb/repo.go                       # DynamoDB (aws-sdk-go-v2)
      memory/repo.go                    # tests / local
      system/system.go                  # SystemClock, ULID generator
      httpapi/                          # driving adapter (Gin)
        router.go  handlers.go  dto.go  identity.go  errors.go
      repotest/contract.go              # shared contract suite (LSP)
```
Allowed imports: `httpapi → app → domain`; `ddb|memory|system → app → domain`; `cmd/api` → all. `domain` imports nothing from the module.

## Domain

```go
// internal/domain/errors.go
package domain

import "errors"

var ErrNotFound = errors.New("not found")

// ValidationError is a business-rule violation (maps to 422 at the edge).
type ValidationError struct{ Msg string }

func (e *ValidationError) Error() string { return e.Msg }
func invalid(msg string) error         { return &ValidationError{Msg: msg} }
```

```go
// internal/domain/money.go
package domain

type Money struct {
	Cents    int64
	Currency string
}

var supported = map[string]bool{"USD": true, "EUR": true, "MXN": true}

// NewMoney is the only way to build a valid Money (value object).
func NewMoney(cents int64, currency string) (Money, error) {
	if currency == "" {
		currency = "USD"
	}
	if cents <= 0 {
		return Money{}, invalid("amount must be a positive number of cents")
	}
	if !supported[currency] {
		return Money{}, invalid("unsupported currency " + currency)
	}
	return Money{Cents: cents, Currency: currency}, nil
}
```

```go
// internal/domain/expense.go
package domain

import (
	"strings"
	"time"
)

type Category string

const (
	Food      Category = "food"
	Transport Category = "transport"
	Housing   Category = "housing"
	Fun       Category = "fun"
	Other     Category = "other"
)

func ParseCategory(s string) (Category, error) {
	switch c := Category(s); c {
	case Food, Transport, Housing, Fun, Other:
		return c, nil
	}
	return "", invalid("unknown category " + s)
}

// Date is a calendar date (no time zone games): midnight UTC.
type Date = time.Time

func ParseDate(s string) (Date, error) {
	d, err := time.Parse("2006-01-02", s)
	if err != nil {
		return Date{}, invalid("date must be YYYY-MM-DD")
	}
	return d, nil
}

type Expense struct {
	ID          string
	UserID      string
	Amount      Money
	Category    Category
	SpentOn     Date
	Description string
	CreatedAt   time.Time
}

// NewExpense enforces invariants. today/now are parameters, so there is no hidden clock.
func NewExpense(id, userID string, amount Money, cat Category, spentOn, today Date, desc string, now time.Time) (Expense, error) {
	if spentOn.After(today) {
		return Expense{}, invalid("date cannot be in the future")
	}
	desc = strings.TrimSpace(desc)
	if len(desc) > 200 {
		return Expense{}, invalid("description too long (max 200)")
	}
	return Expense{ID: id, UserID: userID, Amount: amount, Category: cat, SpentOn: spentOn, Description: desc, CreatedAt: now}, nil
}
```

```go
// internal/domain/summary.go
package domain

import "sort"

type CategoryTotal struct {
	Category Category
	Cents    int64
}

func Summarize(expenses []Expense) []CategoryTotal {
	m := map[Category]int64{}
	for _, e := range expenses {
		m[e.Category] += e.Amount.Cents
	}
	out := make([]CategoryTotal, 0, len(m))
	for c, t := range m {
		out = append(out, CategoryTotal{c, t})
	}
	sort.Slice(out, func(i, j int) bool { return out[i].Cents > out[j].Cents })
	return out
}
```

## Application: ports and use cases

```go
// internal/app/ports.go — interfaces live with their CONSUMER (the use cases)
package app

import (
	"context"
	"time"

	"github.com/YOUR_USER/expense-tracker/api-gin/internal/domain"
)

type ExpenseReader interface {
	Get(ctx context.Context, userID, id string) (domain.Expense, error) // domain.ErrNotFound if absent or not owned
	ListRange(ctx context.Context, userID string, from, to time.Time, limit int, cursor string) (items []domain.Expense, next string, err error)
}

type ExpenseWriter interface {
	Add(ctx context.Context, e domain.Expense) error
	Delete(ctx context.Context, userID, id string) error
}

// ExpenseRepository is for wiring and contract tests only; use cases take the narrow interfaces.
type ExpenseRepository interface {
	ExpenseReader
	ExpenseWriter
}

type Clock interface{ Now() time.Time }
type IDGenerator interface{ NewID() string }
```

```go
// internal/app/create_expense.go
package app

import (
	"context"
	"time"

	"github.com/YOUR_USER/expense-tracker/api-gin/internal/domain"
)

type CreateExpenseInput struct {
	UserID      string
	AmountCents int64
	Currency    string
	Category    string
	SpentOn     string // YYYY-MM-DD
	Description string
}

type CreateExpense struct {
	w     ExpenseWriter
	ids   IDGenerator
	clock Clock
}

func NewCreateExpense(w ExpenseWriter, ids IDGenerator, c Clock) *CreateExpense {
	return &CreateExpense{w: w, ids: ids, clock: c}
}

func (uc *CreateExpense) Execute(ctx context.Context, in CreateExpenseInput) (domain.Expense, error) {
	amount, err := domain.NewMoney(in.AmountCents, in.Currency)
	if err != nil {
		return domain.Expense{}, err
	}
	cat, err := domain.ParseCategory(in.Category)
	if err != nil {
		return domain.Expense{}, err
	}
	spentOn, err := domain.ParseDate(in.SpentOn)
	if err != nil {
		return domain.Expense{}, err
	}
	now := uc.clock.Now().UTC()
	today := now.Truncate(24 * time.Hour)
	e, err := domain.NewExpense(uc.ids.NewID(), in.UserID, amount, cat, spentOn, today, in.Description, now)
	if err != nil {
		return domain.Expense{}, err
	}
	if err := uc.w.Add(ctx, e); err != nil {
		return domain.Expense{}, err
	}
	return e, nil
}
```
```go
// internal/app/list_expenses.go
package app

import (
	"context"
	"time"

	"github.com/YOUR_USER/expense-tracker/api-gin/internal/domain"
)

type ListExpensesInput struct {
	UserID   string
	From, To time.Time
	Limit    int
	Cursor   string
}

type Page struct {
	Items []domain.Expense
	Next  string
}

type ListExpenses struct{ r ExpenseReader }

func NewListExpenses(r ExpenseReader) *ListExpenses { return &ListExpenses{r: r} }

func (uc *ListExpenses) Execute(ctx context.Context, in ListExpensesInput) (Page, error) {
	if in.To.Before(in.From) {
		return Page{}, &domain.ValidationError{Msg: "'to' must not be before 'from'"}
	}
	limit := min(max(in.Limit, 1), 100) // application rule: cap page size (Go 1.21 min/max builtins)
	items, next, err := uc.r.ListRange(ctx, in.UserID, in.From, in.To, limit, in.Cursor)
	return Page{Items: items, Next: next}, err
}
```
`GetExpense`, `DeleteExpense` and `MonthlySummary` follow the same shape (struct with the narrowest port, `Execute` method). `NewX` constructors are the injection points.

## Adapters

### DynamoDB (driven)

```go
// internal/adapters/ddb/repo.go
package ddb

type Repo struct {
	db    *dynamodb.Client
	table string
}

func New(db *dynamodb.Client, table string) *Repo { return &Repo{db: db, table: table} }

// storage model: separate from the domain type, so key design never leaks inward
type record struct {
	PK, SK, GSI1PK, GSI1SK string
	ID                     string `dynamodbav:"id"`
	UserID                 string `dynamodbav:"userId"`
	AmountCents            int64  `dynamodbav:"amountCents"`
	Currency               string `dynamodbav:"currency"`
	Category               string `dynamodbav:"category"`
	SpentOn                string `dynamodbav:"spentOn"`
	Description            string `dynamodbav:"description,omitempty"`
	CreatedAt              string `dynamodbav:"createdAt"`
}

func toRecord(e domain.Expense) record {
	d := e.SpentOn.Format("2006-01-02")
	return record{PK: "USER#" + e.UserID, SK: "EXP#" + e.ID, GSI1PK: "USER#" + e.UserID, GSI1SK: "DATE#" + d + "#" + e.ID,
		ID: e.ID, UserID: e.UserID, AmountCents: e.Amount.Cents, Currency: e.Amount.Currency, Category: string(e.Category),
		SpentOn: d, Description: e.Description, CreatedAt: e.CreatedAt.Format(time.RFC3339Nano)}
}

func (r record) toDomain() (domain.Expense, error) {
	spent, err := domain.ParseDate(r.SpentOn)
	if err != nil {
		return domain.Expense{}, err
	}
	created, err := time.Parse(time.RFC3339Nano, r.CreatedAt)
	if err != nil {
		return domain.Expense{}, err
	}
	return domain.Expense{ID: r.ID, UserID: r.UserID, Amount: domain.Money{Cents: r.AmountCents, Currency: r.Currency},
		Category: domain.Category(r.Category), SpentOn: spent, Description: r.Description, CreatedAt: created}, nil
}

func (r *Repo) Add(ctx context.Context, e domain.Expense) error {
	item, err := attributevalue.MarshalMap(toRecord(e))
	if err != nil {
		return err
	}
	_, err = r.db.PutItem(ctx, &dynamodb.PutItemInput{TableName: &r.table, Item: item,
		ConditionExpression: aws.String("attribute_not_exists(PK)")})
	return err
}

func (r *Repo) Get(ctx context.Context, userID, id string) (domain.Expense, error) {
	out, err := r.db.GetItem(ctx, &dynamodb.GetItemInput{TableName: &r.table, Key: map[string]types.AttributeValue{
		"PK": &types.AttributeValueMemberS{Value: "USER#" + userID},
		"SK": &types.AttributeValueMemberS{Value: "EXP#" + id}}})
	if err != nil {
		return domain.Expense{}, err
	}
	if out.Item == nil {
		return domain.Expense{}, domain.ErrNotFound
	}
	var rec record
	if err := attributevalue.UnmarshalMap(out.Item, &rec); err != nil {
		return domain.Expense{}, err
	}
	return rec.toDomain()
}
// Delete and ListRange: as in infra/04; ListRange encodes LastEvaluatedKey as an opaque base64 JSON string.
```
`r.db` is the concrete AWS client here, fine: this is the outermost ring. For adapter unit tests you can narrow it to a tiny interface of just the DynamoDB calls used.

### In-memory (driven, for tests/local)

```go
// internal/adapters/memory/repo.go
package memory

type Repo struct {
	mu   sync.RWMutex
	data map[string]domain.Expense // key: userID + "|" + id
}

func New() *Repo { return &Repo{data: map[string]domain.Expense{}} }

func (r *Repo) Add(_ context.Context, e domain.Expense) error {
	r.mu.Lock(); defer r.mu.Unlock()
	r.data[e.UserID+"|"+e.ID] = e
	return nil
}
func (r *Repo) Get(_ context.Context, userID, id string) (domain.Expense, error) {
	r.mu.RLock(); defer r.mu.RUnlock()
	e, ok := r.data[userID+"|"+id]
	if !ok {
		return domain.Expense{}, domain.ErrNotFound
	}
	return e, nil
}
// Delete, ListRange (sort by SpentOn desc, ID desc; cursor = base64 of offset)
```

### HTTP (driving): Gin lives only here

```go
// internal/adapters/httpapi/handlers.go
package httpapi

// Consumer-side interfaces: the handler declares exactly what it needs (ISP) — concrete use cases satisfy them.
type expenseCreator interface {
	Execute(context.Context, app.CreateExpenseInput) (domain.Expense, error)
}
type expenseGetter interface {
	Execute(context.Context, string, string) (domain.Expense, error)
}
type expenseLister interface {
	Execute(context.Context, app.ListExpensesInput) (app.Page, error)
}

type Handlers struct {
	create expenseCreator
	get    expenseGetter
	list   expenseLister
}

func NewHandlers(c expenseCreator, g expenseGetter, l expenseLister) *Handlers {
	return &Handlers{create: c, get: g, list: l}
}

func (h *Handlers) Create(c *gin.Context) {
	var req CreateExpenseRequest // Gin binding tags live in dto.go: transport concern only
	if err := c.ShouldBindJSON(&req); err != nil {
		writeProblem(c, http.StatusBadRequest, "bad_request", err.Error())
		return
	}
	e, err := h.create.Execute(c.Request.Context(), app.CreateExpenseInput{
		UserID: c.GetString(ctxUserID), AmountCents: req.AmountCents, Currency: req.Currency,
		Category: req.Category, SpentOn: req.Date, Description: req.Description})
	if err != nil {
		writeError(c, err)
		return
	}
	c.Header("Location", "/expenses/"+e.ID)
	c.JSON(http.StatusCreated, toResponse(e))
}
```

```go
// internal/adapters/httpapi/errors.go — the only place domain errors meet status codes
func writeError(c *gin.Context, err error) {
	var ve *domain.ValidationError
	switch {
	case errors.Is(err, domain.ErrNotFound):
		writeProblem(c, http.StatusNotFound, "not_found", "resource not found")
	case errors.As(err, &ve):
		writeProblem(c, http.StatusUnprocessableEntity, "invalid_expense", ve.Msg)
	default:
		_ = c.Error(err) // logged by middleware; never leak internals
		writeProblem(c, http.StatusInternalServerError, "internal_error", "something went wrong")
	}
}
```
`dto.go` holds `CreateExpenseRequest` (with `binding` tags) and `ExpenseResponse` plus `toResponse(domain.Expense)`. `identity.go` is the `Identity()` middleware from [../infra/04](../infra/04-backend-containers-fastapi-and-gin.md) setting `ctxUserID`.

## Composition root

```go
// cmd/api/main.go
func main() {
	ctx := context.Background()
	cfg, _ := config.LoadDefaultConfig(ctx)
	db := dynamodb.NewFromConfig(cfg)

	repo := ddb.New(db, os.Getenv("TABLE_NAME")) // satisfies app.ExpenseRepository
	clock, ids := system.Clock{}, system.ULID{}

	createUC := app.NewCreateExpense(repo, ids, clock)
	getUC := app.NewGetExpense(repo)
	listUC := app.NewListExpenses(repo)

	router := httpapi.NewRouter(httpapi.NewHandlers(createUC, getUC, listUC))
	srv := &http.Server{Addr: ":8080", Handler: router, ReadHeaderTimeout: 5 * time.Second}
	// ...graceful shutdown as in 02
}
```
Swap `ddb.New(...)` for `memory.New()` to run with zero AWS; swap in a Postgres adapter later; nothing else changes.

## Tests

**Domain**: table-driven, no mocks.
```go
func TestNewExpense_FutureDate(t *testing.T) {
	today := time.Date(2026, 10, 6, 0, 0, 0, 0, time.UTC)
	_, err := domain.NewExpense("1", "u", domain.Money{Cents: 100, Currency: "USD"}, domain.Food,
		today.AddDate(0, 0, 1), today, "", today)
	var ve *domain.ValidationError
	if !errors.As(err, &ve) { t.Fatalf("want ValidationError, got %v", err) }
}
```

**Use case**: fakes defined in the test file (tiny, consumer-shaped).
```go
type fixedClock struct{ t time.Time }
func (f fixedClock) Now() time.Time { return f.t }
type seqIDs struct{ n int }
func (s *seqIDs) NewID() string { s.n++; return fmt.Sprintf("e%d", s.n) }

func TestCreateExpense_Persists(t *testing.T) {
	repo := memory.New()
	uc := app.NewCreateExpense(repo, &seqIDs{}, fixedClock{time.Date(2026, 10, 6, 12, 0, 0, 0, time.UTC)})
	e, err := uc.Execute(context.Background(), app.CreateExpenseInput{UserID: "u1", AmountCents: 1250, Category: "food", SpentOn: "2026-10-05"})
	if err != nil { t.Fatal(err) }
	got, err := repo.Get(context.Background(), "u1", e.ID)
	if err != nil || got.Amount.Cents != 1250 { t.Fatalf("got %+v, %v", got, err) }
}
```

**Contract suite (LSP)**: one function, every adapter:
```go
// internal/adapters/repotest/contract.go
func Run(t *testing.T, newRepo func(t *testing.T) app.ExpenseRepository) {
	t.Run("other user cannot see expense", func(t *testing.T) {
		r := newRepo(t)
		_ = r.Add(ctx, sample("alice", "e1", "2026-10-01"))
		if _, err := r.Get(ctx, "bob", "e1"); !errors.Is(err, domain.ErrNotFound) {
			t.Fatalf("want ErrNotFound, got %v", err)
		}
	})
	t.Run("list is newest first and paginates without gaps", func(t *testing.T) { /* add 5, page by 3, compare IDs */ })
}

// memory/repo_test.go:  func TestContract(t *testing.T) { repotest.Run(t, func(*testing.T) app.ExpenseRepository { return memory.New() }) }
// ddb/repo_test.go:     skips unless DYNAMODB_ENDPOINT is set; creates a uniquely named table per test run.
```

**HTTP adapter**: `httptest` against the router built from use cases wired to `memory.New()` (see [02](02-gin-rest-api.md) for the recorder pattern).

## Enforce the layers

```yaml
# .golangci.yml  (golangci-lint)
linters: { enable: [depguard, revive] }
linters-settings:
  depguard:
    rules:
      domain-is-pure:
        files: ["**/internal/domain/**"]
        deny:
          - { pkg: "github.com/YOUR_USER/expense-tracker/api-gin/internal/app", desc: "domain must not depend on application" }
          - { pkg: "github.com/YOUR_USER/expense-tracker/api-gin/internal/adapters", desc: "domain must not depend on adapters" }
          - { pkg: "github.com/gin-gonic/gin", desc: "no frameworks in domain" }
          - { pkg: "github.com/aws/aws-sdk-go-v2", desc: "no SDKs in domain" }
      app-is-framework-free:
        files: ["**/internal/app/**"]
        deny:
          - { pkg: "github.com/YOUR_USER/expense-tracker/api-gin/internal/adapters", desc: "application must not depend on adapters" }
          - { pkg: "github.com/gin-gonic/gin", desc: "no frameworks in application" }
          - { pkg: "github.com/aws/aws-sdk-go-v2", desc: "no SDKs in application" }
```
Run in CI next to `go vet`, `go test -race`, `govulncheck` ([05](05-deploy-and-testing.md)).

## Mapping of the earlier Go tutorials
| Earlier | Now |
|---|---|
| 01 interfaces, errors, `context` | the language features this layout relies on |
| 02 handlers/service/repo | `adapters/httpapi` / `app` / `adapters/*` (handler no longer holds business logic) |
| 03 concurrency (`errgroup`) | inside use cases that fan out (e.g. dashboard) with `context`, still behind ports |
| 04 pgx repo + JWT | an additional adapter `adapters/pg` (contract suite applies) and `httpapi` identity middleware |
| 05 deployment | unchanged: only `cmd/api/main.go` wires concrete types |

## SOLID in Go, concretely
| Principle | In this code |
|---|---|
| SRP | `domain` = rules, `app` = intents, `httpapi` = translation, `ddb` = persistence |
| OCP | new `adapters/pg` or `adapters/bedrock` + one line in `main.go` |
| LSP | `repotest.Run` passes for memory and ddb |
| ISP | handlers declare 1-method interfaces; use cases take `ExpenseWriter`, not a god repo |
| DIP | `app` defines `ExpenseReader`; `ddb.Repo` satisfies it without importing `app` interfaces; wired in `main.go` |

## Exercise
1. Implement `GetExpense`, `DeleteExpense`, `MonthlySummary` and their tests.
2. Write `repotest.Run` fully and run it on memory and DynamoDB Local.
3. Add the `depguard` rules and break them on purpose.
4. Add `adapters/bedrock` implementing an `InsightGenerator` port defined in `app`; fake it in tests.

## Interview Q&A
- **Why define interfaces at the consumer in Go?** Keeps the dependency pointing inward and interfaces minimal; implementers don't need to know the interface exists.
- **Where do struct tags go?** Binding/JSON tags on transport DTOs and `dynamodbav` tags on storage records, **not** on domain types.
- **How do you do dependency injection without a framework?** Constructors + a `main.go` composition root (Wire/Fx exist but are optional).
- **How do you prevent import cycles?** The layering is acyclic by design; adapters depend on app/domain, never the reverse.
- **How do you test the HTTP layer without a DB?** Build the router from real use cases over `memory.Repo`.
