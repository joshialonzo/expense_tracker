# Go 06 — Interview Questions and Drills

Since Go is "nice to have", expect fewer and shallower questions: fundamentals, concurrency, error handling, and a small REST service.

## Concept questions
1. What makes Go different from Python/Java/Node? (compiled, static, simple, goroutines, composition, single binary)
2. Explain slices vs arrays; what happens on `append`; aliasing gotcha.
3. Maps: iteration order, nil map write, concurrent access.
4. Interfaces: implicit implementation; empty interface; the nil-interface-with-nil-pointer gotcha.
5. Pointers vs values; method receivers; when to use each.
6. Error handling: `errors.Is/As`, wrapping with `%w`, sentinel vs typed errors; why no exceptions.
7. `defer`, `panic`, `recover`: semantics and uses.
8. Goroutines, channels (buffered/unbuffered), `select`, `sync.WaitGroup`, `Mutex`, `context` ([03](03-concurrency.md)).
9. How does Go's scheduler work (G, M, P)? (Goroutines multiplexed onto OS threads via per-P run queues; work stealing; I/O parks goroutines.)
10. Garbage collection: concurrent mark-sweep, tune with `GOGC`/`GOMEMLIMIT`; reduce allocations (preallocate slices, `sync.Pool`).
11. Generics: when useful (containers, `Map/Filter`) vs not (don't over-abstract).
12. Packages/modules: `go.mod`, `internal/`, semantic import versioning (`/v2`).
13. How do you test in Go? table-driven tests, `httptest`, `testify`, fakes via interfaces, `-race`.
14. What are struct tags? (`json:"name,omitempty"`, `binding:"required"`)
15. `init()` order, package-level state: avoid.

## Drills (write on the board / editor)

### 1. Reverse words, count frequencies
```go
func wordFreq(s string) map[string]int {
	m := make(map[string]int)
	for _, w := range strings.Fields(strings.ToLower(s)) { m[w]++ }
	return m
}
```

### 2. Thread-safe counter / cache
Use `sync.Mutex`/`RWMutex` with a struct; add TTL eviction; test with `-race`.

### 3. Concurrent fetch with timeout
Fetch N URLs concurrently with `errgroup`, bounded to 5, honoring `ctx` ([03](03-concurrency.md)).

### 4. Fix the bug
```go
func process(items []Item) {
	var wg sync.WaitGroup
	for _, it := range items {
		go func() {          // before Go 1.22: captures loop var; also missing wg.Add/Done
			handle(it)
		}()
	}
	wg.Wait()
}
```

### 5. Middleware
Write a Gin middleware that rate limits by user ID (token bucket via `x/time/rate`) and returns `429` with `Retry-After`.

### 6. LRU cache
Doubly linked list (`container/list`) + map; O(1) get/put.

### 7. Parse and validate JSON
Decode with `json.NewDecoder(r.Body)`; `DisallowUnknownFields()`; limit body size with `http.MaxBytesReader`.

## "Design with Go" prompts
- Receipt-processing worker consuming SQS with at-least-once delivery: visibility timeout, idempotent processing, worker pool, graceful shutdown, DLQ.
- A high-throughput API gateway in front of the MCP/AI services: timeouts, circuit breaker, per-user rate limits, streaming proxying (`io.Copy`/`http.Flusher`).

## Go vs Python (talk track)
See table in [../fastapi/06-interview-questions.md](../fastapi/06-interview-questions.md). One-liner: "Go for throughput, latency, small deploy artifacts, concurrency-heavy infra services; Python for AI/data work and fast iteration; contracts in OpenAPI keep either swappable."

## Study plan if you're new to Go (3 evenings)
1. **Tour of Go** (go.dev/tour) + *Effective Go* skim; write the code in [01](01-go-fundamentals.md).
2. Build the REST service ([02](02-gin-rest-api.md)) with an in-memory repo and tests.
3. Add Postgres + JWT ([04](04-database-and-auth.md)) and concurrency exercises ([03](03-concurrency.md)); containerize ([05](05-deploy-and-testing.md)).
Be honest in the interview: "I'm strongest in X; I've built a Go service with Gin, Postgres, JWT and deployed it on ECS/Lambda, and I'm comfortable with goroutines, contexts and error handling."
