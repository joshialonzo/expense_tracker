# Go 02 — Building the Expense REST API with Gin

## Setup

```bash
go get github.com/gin-gonic/gin
go get github.com/google/uuid
```

> **The clean-architecture version of this layout is in [07](07-clean-architecture.md)** (`internal/domain`, `internal/app`, `internal/adapters/{httpapi,ddb,memory}`, composition root in `cmd/api/main.go`). Treat the structure below as the stepping stone: handlers here call a service; in 07 they call single-purpose use cases through consumer-defined interfaces.

## Layout (standard-ish, idiomatic)

```
api-gin/
  cmd/api/main.go            # wiring, config, server start, graceful shutdown
  internal/
    config/config.go
    http/                    # handlers, router, middleware
      router.go
      expenses_handler.go
      middleware.go
    expenses/                # domain: types, service, repository interface
      model.go
      service.go
      repo_pg.go
    auth/jwt.go
  migrations/
  go.mod
```
`internal/` prevents outside modules from importing your packages. Packages by **feature/domain**, not by technical layer type.

## Domain + service

```go
// internal/expenses/model.go
package expenses

import "time"

type Expense struct {
	ID          string    `json:"id"`
	UserID      string    `json:"-"`                     // never serialized
	AmountCents int64     `json:"amountCents"`
	Currency    string    `json:"currency"`
	Category    string    `json:"category"`
	Description *string   `json:"description,omitempty"`
	SpentOn     string    `json:"date"`                   // YYYY-MM-DD
	CreatedAt   time.Time `json:"createdAt"`
}

type CreateInput struct {
	AmountCents int64   `json:"amountCents" binding:"required,gt=0"`
	Currency    string  `json:"currency"    binding:"omitempty,len=3,uppercase"`
	Category    string  `json:"category"    binding:"required,oneof=food transport housing fun other"`
	Description *string `json:"description" binding:"omitempty,max=200"`
	SpentOn     string  `json:"date"        binding:"required,datetime=2006-01-02"`
}
```
`binding:"..."` tags use the `go-playground/validator` library. Separate input structs (no `ID`, `UserID`) prevent mass-assignment.

```go
// internal/expenses/service.go
package expenses

import (
	"context"
	"errors"
)

var ErrNotFound = errors.New("expense not found")

type Repo interface {
	Create(ctx context.Context, userID string, in CreateInput) (Expense, error)
	Get(ctx context.Context, userID, id string) (Expense, error)       // returns ErrNotFound if not owned/missing
	List(ctx context.Context, userID string, f ListFilter) ([]Expense, string, error)
	Delete(ctx context.Context, userID, id string) error
}

type Service struct{ repo Repo }

func NewService(r Repo) *Service { return &Service{repo: r} }

func (s *Service) Create(ctx context.Context, userID string, in CreateInput) (Expense, error) {
	// business rules here (e.g. no future dates)
	return s.repo.Create(ctx, userID, in)
}
// Get/List/Delete delegate similarly
```

## Handlers

```go
// internal/http/expenses_handler.go
package http

import (
	"errors"
	"net/http"

	"github.com/gin-gonic/gin"
	"github.com/YOUR_USER/expense-tracker/api-gin/internal/expenses"
)

type ExpensesHandler struct{ svc *expenses.Service }

func (h *ExpensesHandler) Register(r gin.IRoutes) {
	r.GET("/expenses", h.list)
	r.POST("/expenses", h.create)
	r.GET("/expenses/:id", h.get)
	r.DELETE("/expenses/:id", h.delete)
}

func (h *ExpensesHandler) create(c *gin.Context) {
	var in expenses.CreateInput
	if err := c.ShouldBindJSON(&in); err != nil {           // ShouldBind* returns the error; Bind* aborts with 400
		problem(c, http.StatusUnprocessableEntity, "validation_error", err.Error())
		return
	}
	userID := c.GetString("userID")                          // set by auth middleware
	e, err := h.svc.Create(c.Request.Context(), userID, in)
	if err != nil {
		h.fail(c, err)
		return
	}
	c.Header("Location", "/expenses/"+e.ID)
	c.JSON(http.StatusCreated, e)
}

func (h *ExpensesHandler) get(c *gin.Context) {
	e, err := h.svc.Get(c.Request.Context(), c.GetString("userID"), c.Param("id"))
	if err != nil {
		h.fail(c, err)
		return
	}
	c.JSON(http.StatusOK, e)
}

func (h *ExpensesHandler) delete(c *gin.Context) {
	if err := h.svc.Delete(c.Request.Context(), c.GetString("userID"), c.Param("id")); err != nil {
		h.fail(c, err)
		return
	}
	c.Status(http.StatusNoContent)
}

func (h *ExpensesHandler) fail(c *gin.Context, err error) {
	switch {
	case errors.Is(err, expenses.ErrNotFound):
		problem(c, http.StatusNotFound, "not_found", "expense not found")
	default:
		c.Error(err) // logged by middleware
		problem(c, http.StatusInternalServerError, "internal_error", "something went wrong")
	}
}

func problem(c *gin.Context, status int, code, detail string) {
	c.AbortWithStatusJSON(status, gin.H{"type": "https://api.example.com/problems/" + code, "status": status, "detail": detail})
}
```

Query params with binding:

```go
type ListQuery struct {
	From     string `form:"from"     binding:"omitempty,datetime=2006-01-02"`
	To       string `form:"to"       binding:"omitempty,datetime=2006-01-02"`
	Category string `form:"category"`
	Limit    int    `form:"limit,default=50" binding:"min=1,max=100"`
	Cursor   string `form:"cursor"`
}
func (h *ExpensesHandler) list(c *gin.Context) {
	var q ListQuery
	if err := c.ShouldBindQuery(&q); err != nil { /* 400 */ return }
	...
}
```

## Router, middleware, server with graceful shutdown

```go
// internal/http/router.go
func NewRouter(h *ExpensesHandler, verify auth.Verifier, log *slog.Logger) *gin.Engine {
	r := gin.New()
	r.Use(gin.Recovery(), RequestID(), AccessLog(log), cors.New(cors.Config{
		AllowOrigins: []string{"http://localhost:5173", "https://app.example.com"},
		AllowMethods: []string{"GET", "POST", "PATCH", "DELETE"},
		AllowHeaders: []string{"Authorization", "Content-Type", "Idempotency-Key"},
	}))

	r.GET("/health", func(c *gin.Context) { c.JSON(200, gin.H{"status": "ok"}) })

	api := r.Group("/", auth.Required(verify)) // everything below needs a valid JWT
	h.Register(api)
	return r
}
```

```go
// middleware.go
func RequestID() gin.HandlerFunc {
	return func(c *gin.Context) {
		id := c.GetHeader("X-Request-Id")
		if id == "" { id = uuid.NewString() }
		c.Set("requestID", id)
		c.Header("X-Request-Id", id)
		c.Next()
	}
}

func AccessLog(log *slog.Logger) gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()
		c.Next() // run the rest of the chain
		log.Info("request", "method", c.Request.Method, "path", c.FullPath(), "status", c.Writer.Status(),
			"ms", time.Since(start).Milliseconds(), "rid", c.GetString("requestID"))
	}
}
```
`c.Next()` calls downstream handlers; `c.Abort*` stops the chain; `c.Set/Get` pass values (use typed helpers in real code).

```go
// cmd/api/main.go
func main() {
	log := slog.New(slog.NewJSONHandler(os.Stdout, nil))
	cfg := config.Load()
	db := mustPool(cfg.DatabaseURL)
	defer db.Close()

	svc := expenses.NewService(expenses.NewPgRepo(db))
	router := apphttp.NewRouter(apphttp.NewExpensesHandler(svc), auth.NewCognitoVerifier(cfg), log)

	srv := &http.Server{Addr: ":8080", Handler: router, ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout: 15 * time.Second, WriteTimeout: 30 * time.Second, IdleTimeout: 60 * time.Second}

	go func() {
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) { log.Error("server", "err", err); os.Exit(1) }
	}()

	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()
	<-ctx.Done() // ECS sends SIGTERM on deploys
	shutdownCtx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
	defer cancel()
	_ = srv.Shutdown(shutdownCtx) // stop accepting, drain in-flight requests
}
```
Always set **server timeouts** (slowloris protection) and use `gin.SetMode(gin.ReleaseMode)` in prod.

## Unit-testing handlers (`httptest`)

```go
func TestCreateExpense_Validation(t *testing.T) {
	gin.SetMode(gin.TestMode)
	r := gin.New()
	h := &ExpensesHandler{svc: expenses.NewService(&fakeRepo{})}
	r.Use(func(c *gin.Context) { c.Set("userID", "u1") })
	h.Register(r)

	body := strings.NewReader(`{"amountCents": -1, "category": "food", "date": "2026-10-01"}`)
	req := httptest.NewRequest(http.MethodPost, "/expenses", body)
	req.Header.Set("Content-Type", "application/json")
	w := httptest.NewRecorder()
	r.ServeHTTP(w, req)

	if w.Code != http.StatusUnprocessableEntity { t.Fatalf("got %d: %s", w.Code, w.Body) }
}
```

## OpenAPI for the contract
Generate docs with `swaggo/swag` (annotations → `swagger.json`) or write the spec first and generate server stubs with `oapi-codegen` (supports Gin). The same contract lets the React and Angular clients generate types ([../fullstack/01](../fullstack/01-rest-api-design.md)).

## Gin vs standard library vs others
Go 1.22+ `net/http` has method+path patterns (`mux.HandleFunc("GET /expenses/{id}", …)`), so a framework is optional. Gin adds binding/validation, middleware chaining, JSON helpers and good performance. Alternatives: Echo, Fiber (fasthttp, non-standard), chi (stdlib-compatible, minimal). Say you'd pick based on team familiarity and keep handlers thin so switching is cheap.

## Exercise
Implement all six endpoints with an in-memory repo first, then table-driven handler tests, then plug the Postgres repo ([04](04-database-and-auth.md)). Verify graceful shutdown by sending `SIGTERM` during a slow request.

## Interview Q&A
- **`ShouldBind` vs `Bind`?** `Should*` returns the error so you control the response; `Bind*` writes a 400 and aborts.
- **How does Gin middleware work?** Chain of handlers; `Next()` continues, `Abort()` stops; code after `Next()` runs on the way back.
- **Why `c.Request.Context()`?** Propagate cancellation/timeouts to downstream calls.
- **How do you shut down gracefully?** `http.Server.Shutdown(ctx)` on SIGTERM.
- **How do you avoid exposing internal fields?** Separate DTOs, `json:"-"`.
