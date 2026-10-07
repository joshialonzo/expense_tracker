# Go 03 — Concurrency: Goroutines, Channels, sync and Patterns

## Why it matters
Concurrency is *the* Go interview topic. Know the primitives, the classic bugs, and a few patterns.

## Goroutines
Lightweight (KBs of stack, multiplexed on OS threads by the runtime scheduler). Start with `go f()`. They're not free: leaked goroutines leak memory.

```go
go func() { sendEmail(ctx, user) }()   // fire and forget (who waits? who handles errors? usually a bug)
```
`main` exiting kills all goroutines.

## Channels

```go
ch := make(chan int)        // unbuffered: send blocks until a receiver is ready (sync point)
bch := make(chan int, 10)   // buffered: send blocks only when full

go func() { defer close(ch); for i := 0; i < 3; i++ { ch <- i } }()
for v := range ch { fmt.Println(v) }   // range until closed

v, ok := <-ch                          // ok == false when closed and drained
```
Rules: **only the sender closes**; sending on a closed channel panics; receiving from a nil channel blocks forever. Prefer channels for **ownership transfer/pipelines**, mutexes for **protecting shared state**. ("Don't communicate by sharing memory; share memory by communicating", but use a mutex when it's simpler.)

### select

```go
select {
case res := <-results:
	handle(res)
case <-ctx.Done():
	return ctx.Err()          // cancellation/timeouts
case <-time.After(2 * time.Second):
	return errors.New("timeout")
default:                       // non-blocking
}
```

## sync package

```go
var mu sync.Mutex             // or sync.RWMutex for read-heavy
type Cache struct { mu sync.RWMutex; m map[string]Summary }
func (c *Cache) Get(k string) (Summary, bool) { c.mu.RLock(); defer c.mu.RUnlock(); v, ok := c.m[k]; return v, ok }
func (c *Cache) Set(k string, v Summary)      { c.mu.Lock(); defer c.mu.Unlock(); c.m[k] = v }

var wg sync.WaitGroup
for _, id := range ids {
	wg.Add(1)
	go func() { defer wg.Done(); process(id) }()   // Go 1.22+: loop var is per-iteration, so capture is safe
}
wg.Wait()

var once sync.Once; once.Do(initClient)   // lazy singleton init
```
Also `sync/atomic` for counters, `sync.Pool` for reuse, `singleflight` (golang.org/x/sync) to collapse duplicate concurrent calls (cache stampede protection).

## errgroup: the practical workhorse

```go
import "golang.org/x/sync/errgroup"

func Dashboard(ctx context.Context, svc *Service, userID string) (*Dash, error) {
	g, ctx := errgroup.WithContext(ctx)   // first error cancels ctx for the others
	var expenses []Expense
	var summary Summary

	g.Go(func() (err error) { expenses, err = svc.Recent(ctx, userID); return })
	g.Go(func() (err error) { summary, err = svc.MonthSummary(ctx, userID); return })

	if err := g.Wait(); err != nil { return nil, err }
	return &Dash{expenses, summary}, nil
}
```
`g.SetLimit(n)` bounds concurrency.

## Patterns

### Worker pool (bounded concurrency)

```go
func processReceipts(ctx context.Context, keys []string, workers int) error {
	jobs := make(chan string)
	g, ctx := errgroup.WithContext(ctx)

	for i := 0; i < workers; i++ {
		g.Go(func() error {
			for k := range jobs {
				if err := ocr(ctx, k); err != nil { return err }
			}
			return nil
		})
	}
	g.Go(func() error {
		defer close(jobs)
		for _, k := range keys {
			select {
			case jobs <- k:
			case <-ctx.Done():
				return ctx.Err()
			}
		}
		return nil
	})
	return g.Wait()
}
```

### Fan-out/fan-in, pipelines, rate limiting
`golang.org/x/time/rate` limiter per user; `time.Ticker` for scheduling; pipeline stages connected by channels where each stage closes its output when its input closes.

## Classic bugs (they will ask you to spot them)
1. **Data race**: concurrent read/write of a map or variable without sync → run `go test -race`.
2. **Goroutine leak**: goroutine blocked forever on a channel nobody reads/closes; always provide a cancellation path (`ctx`).
3. **Deadlock**: all goroutines asleep (`fatal error: all goroutines are asleep`); mutex lock-ordering; unbuffered send with no receiver.
4. **Loop variable capture** (pre-1.22): closures all see the last value.
5. **Forgetting `wg.Add` before starting the goroutine**, or copying a `sync.Mutex`/`WaitGroup` by value.
6. **Closing a channel twice / sending on closed**.
7. **Unbounded goroutine creation** per request → OOM; bound with pools/semaphores.

Spot the bug:
```go
var count int
for i := 0; i < 1000; i++ { go func() { count++ }() }
time.Sleep(time.Second)
fmt.Println(count)   // race + sleeping instead of synchronizing; fix with atomic/mutex + WaitGroup
```

## Go concurrency model vs async/await vs threads
Go: M:N scheduler, goroutines + blocking-style code, the runtime parks goroutines on I/O. Node/Python asyncio: single-threaded event loop with cooperative `await` (a blocking call stalls everything). OS threads: heavier, preemptive. Go lets you write simple sequential code that scales to 100k concurrent connections.

## Using it in the API
- Per-request goroutines are already provided by `net/http`. Use `errgroup` to parallelize independent downstream calls.
- Background work (email, OCR) → durable queue (SQS) rather than naked goroutines that die on deploy.
- Pass `ctx` everywhere; respect cancellation.
- Protect shared in-memory caches with `RWMutex`/`sync.Map`; for a real cache use Redis.

## Exercise
1. Fetch 50 receipts' metadata with a worker pool of 5; stop on first error.
2. Introduce a race, catch it with `-race`, fix it.
3. Build an in-memory rate limiter per user ID with `x/time/rate` and a janitor goroutine that evicts idle limiters, with clean shutdown.

## Interview Q&A
- **Goroutine vs thread?** Cheap user-space, scheduled by the Go runtime, growable stacks.
- **Buffered vs unbuffered channel?** Sync handoff vs decoupled up to capacity.
- **How do you stop a goroutine?** Context cancellation or a `done` channel.
- **Mutex vs channel?** Mutex for protecting state, channels for coordination/ownership transfer.
- **What does `-race` do?** Instruments memory accesses to detect unsynchronized concurrent access at runtime (not a proof of absence).
- **What is a goroutine leak and how to find it?** Blocked forever; `pprof` goroutine profile, `goleak` in tests.
