# Go 01 — Go Language Fundamentals (Nice-to-Have, but Learn the Core)

## Why Go
Compiled, statically typed, simple, fast, with built-in concurrency and a single static binary: popular for cloud/backend services, CLIs and infra tools. The JD lists Go as nice-to-have, so aim for **idiomatic fundamentals and being able to read/write a small service**.

## Setup

```bash
brew install go              # or download from go.dev
mkdir api-gin && cd api-gin
go mod init github.com/YOUR_USER/expense-tracker/api-gin
go run .                     # compile+run
go build -o bin/api .        # single binary
go test ./...                # all tests
go vet ./... && gofmt -l .   # static checks, formatting (gofmt is non-negotiable)
```

## Core syntax tour

```go
package main

import (
	"errors"
	"fmt"
	"sort"
	"strings"
	"time"
)

// Types
type Category string

const (
	Food      Category = "food"
	Transport Category = "transport"
)

type Expense struct {
	ID          string    `json:"id"`
	AmountCents int64     `json:"amountCents"`
	Currency    string    `json:"currency"`
	Category    Category  `json:"category"`
	Description *string   `json:"description,omitempty"` // pointer = optional/nullable
	SpentOn     time.Time `json:"date"`
}

// Methods: receiver is value or pointer (use pointer if mutating or struct is large)
func (e Expense) Dollars() string { return fmt.Sprintf("%.2f", float64(e.AmountCents)/100) }

// Functions return multiple values; errors are values, not exceptions
func parseCategory(s string) (Category, error) {
	switch c := Category(strings.ToLower(s)); c {
	case Food, Transport:
		return c, nil
	default:
		return "", fmt.Errorf("unknown category %q", s)
	}
}

func totals(expenses []Expense) map[Category]int64 {
	out := make(map[Category]int64)
	for _, e := range expenses { // range over slice: index, value
		out[e.Category] += e.AmountCents
	}
	return out
}

func main() {
	list := []Expense{{ID: "1", AmountCents: 1250, Category: Food}, {ID: "2", AmountCents: 4500, Category: Transport}}
	t := totals(list)
	keys := make([]Category, 0, len(t))
	for k := range t {
		keys = append(keys, k)
	}
	sort.Slice(keys, func(i, j int) bool { return t[keys[i]] > t[keys[j]] })
	for _, k := range keys {
		fmt.Printf("%-10s %d\n", k, t[k])
	}

	if _, err := parseCategory("rent"); err != nil {
		fmt.Println("error:", err)
	}
}
```

## Things to know cold
- **Zero values**: `0`, `""`, `false`, `nil` for pointers/slices/maps/interfaces. Useful defaults; a `nil` map panics on write (`make` it), a `nil` slice is fine to `append`.
- **Slices** (`[]T`: pointer + len + cap; `append` may reallocate; sub-slices share memory) vs arrays (fixed). **Maps**: unordered, not safe for concurrent writes.
- **Pointers**: `&x`, `*p`; no pointer arithmetic. Pass by value by default; pointers for mutation/large structs/optional fields.
- **`:=`** short declaration inside functions; **exported** identifiers start with an uppercase letter.
- **No classes/inheritance**: structs + methods + interfaces + embedding (composition).
- **`defer`**: runs at function exit (LIFO) → cleanup (`defer rows.Close()`, `defer mu.Unlock()`).
- **Packages**: one directory = one package; `internal/` restricts imports to the module.
- **Generics** (1.18+): `func Map[T, U any](xs []T, f func(T) U) []U`.

## Interfaces (implicit satisfaction)

```go
type ExpenseRepo interface {
	Add(ctx context.Context, e Expense) error
	List(ctx context.Context, userID string, from, to time.Time) ([]Expense, error)
}
```
Any type with those methods satisfies it, with no `implements` keyword. Idiom: **accept interfaces, return structs**; define interfaces where they're *used* (consumer side), keep them small (1–3 methods). `any` = `interface{}`. Type assertion `v, ok := x.(T)`, type switch `switch v := x.(type)`.

## Error handling

```go
var ErrNotFound = errors.New("not found")

type ValidationError struct{ Field, Msg string }
func (e *ValidationError) Error() string { return e.Field + ": " + e.Msg }

func (s *Service) Get(ctx context.Context, id string) (Expense, error) {
	e, err := s.repo.Get(ctx, id)
	if err != nil {
		return Expense{}, fmt.Errorf("get expense %s: %w", id, err) // wrap with %w
	}
	return e, nil
}

// caller
if errors.Is(err, ErrNotFound) { /* 404 */ }
var ve *ValidationError
if errors.As(err, &ve) { /* 422 */ }
```
Always check errors; wrap with context; don't `panic` for expected failures (panics are for programmer errors; `recover` at boundaries like HTTP middleware).

## `context.Context`
Carries **deadlines, cancellation and request-scoped values** across API boundaries. First parameter by convention: `func (r *Repo) List(ctx context.Context, ...)`. Always pass the request context down to DB/HTTP calls so they cancel when the client disconnects or the timeout hits.

```go
ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
defer cancel()
rows, err := db.QueryContext(ctx, `SELECT ...`)
```

## Testing is built in

```go
// totals_test.go
func TestTotals(t *testing.T) {
	cases := []struct {
		name string
		in   []Expense
		want map[Category]int64
	}{
		{"empty", nil, map[Category]int64{}},
		{"two", []Expense{{Category: Food, AmountCents: 5}, {Category: Food, AmountCents: 7}}, map[Category]int64{Food: 12}},
	}
	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			if got := totals(tc.in); !reflect.DeepEqual(got, tc.want) {
				t.Fatalf("got %v want %v", got, tc.want)
			}
		})
	}
}
```
Table-driven tests + `t.Run` subtests are idiomatic. Also: `go test -race ./...` (race detector), `-cover`, benchmarks (`BenchmarkX`), fuzzing.

## Exercise
Implement `Money` (cents + currency) with `Add` returning an error on currency mismatch, a `GroupByCategory` generic function, and table-driven tests. Run `go vet`, `gofmt`, `go test -race -cover`.

## Interview Q&A
- **Slice vs array? What does `append` do?** Slice = view (ptr/len/cap) over an array; append reuses capacity or allocates a bigger array and copies.
- **Value vs pointer receivers?** Pointer to mutate/avoid copy; be consistent per type.
- **How does Go handle errors?** Returned values, wrapped with `%w`, inspected with `errors.Is/As`.
- **Interfaces: nil interface gotcha?** An interface holding a nil pointer is **not** `== nil`.
- **What is `defer` evaluation order?** Arguments evaluated at the defer statement, calls run LIFO.
- **Why `context`?** Cancellation/deadline propagation and request-scoped values.
