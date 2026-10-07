# FastAPI / Python 06 — Interview Questions and Drills

## FastAPI
1. How does FastAPI use type hints? (validation, serialization, docs)
2. `def` vs `async def` endpoints; what happens if you block the event loop?
3. How does dependency injection work? Override it in tests?
4. Request vs response models; why `response_model`?
5. How do you handle auth (JWT/OAuth2) and authorization?
6. Background tasks vs a real queue (Celery/SQS)? (`BackgroundTasks` runs in-process after the response; lost if the process dies, so use a queue for important work.)
7. Middleware vs dependencies vs exception handlers; use cases for each.
8. WebSockets/SSE in FastAPI.
9. How do you structure a large FastAPI app? (routers, services, repos, settings)
10. How do you version and document the API? (OpenAPI, `/v1` prefix routers)

## Python language
1. List vs tuple vs set vs dict; hashability; complexity of operations.
2. Generators and `yield`; iterators vs iterables; memory benefit.
3. Decorators (write `@timed`), context managers (`with`, `contextlib`).
4. GIL: what is it, what does it mean for threads vs processes vs asyncio?
5. `asyncio`: event loop, `await`, `gather`, tasks, cancellation, why not mix blocking code.
6. Mutable default args pitfall (`def f(x=[])`), late-binding closures.
7. `dataclasses` vs Pydantic vs `TypedDict` vs `attrs`.
8. Typing: `Protocol`, `TypeVar`, `Generic`, `Literal`, `Annotated`, mypy/pyright.
9. Packaging/envs: venv, pip, poetry/uv, `requirements.txt` pinning.
10. Testing: pytest fixtures, parametrization, monkeypatch, mocking vs fakes.

### Drills
**Decorator**
```python
import functools, time
def timed(fn):
    @functools.wraps(fn)
    def wrapper(*a, **kw):
        t = time.perf_counter()
        try: return fn(*a, **kw)
        finally: print(f"{fn.__name__} {1000*(time.perf_counter()-t):.1f}ms")
    return wrapper
```
**Group expenses**
```python
from collections import defaultdict
def totals_by_category(expenses):
    out = defaultdict(int)
    for e in expenses: out[e["category"]] += e["amount_cents"]
    return dict(sorted(out.items(), key=lambda kv: -kv[1]))
```
**Async fan-out with limit**
```python
import asyncio, httpx
async def fetch_all(urls, limit=10):
    sem = asyncio.Semaphore(limit)
    async with httpx.AsyncClient(timeout=5) as c:
        async def one(u):
            async with sem:
                return (await c.get(u)).json()
        return await asyncio.gather(*(one(u) for u in urls), return_exceptions=True)
```
**Rate limiter (token bucket)** and **LRU cache** ([../fullstack/07](../fullstack/07-interview-questions.md)).

## Architecture scenarios
- "Add CSV import of 100k rows": streaming upload to S3 → SQS → worker with batched `INSERT … ON CONFLICT DO NOTHING`, progress endpoint.
- "The endpoint is slow": profile (`py-spy`, logs with timings), `EXPLAIN`, N+1, payload size, caching, async I/O.
- "Make it idempotent": `Idempotency-Key` table.

## Python vs Go (they accept either, be ready to compare)
| | Python/FastAPI | Go/Gin |
|---|---|---|
| Speed | Good enough for most CRUD | Much faster, low memory |
| Concurrency | asyncio / threads (GIL) | goroutines/channels |
| Types | Optional hints (runtime via Pydantic) | Static, compiled |
| Deploy | Needs runtime/deps (containers/Lambda layers) | Single static binary, tiny images |
| Ecosystem | AI/data/ML libs (boto3, pydantic, pandas) | Cloud/infra tooling |
| Cold start (Lambda) | Hundreds of ms | Tens of ms |
Answer pattern: "For AI-heavy or data-heavy services I'd pick Python; for high-throughput, low-latency gateways or CLIs I'd pick Go; I keep the contract in OpenAPI so the choice is reversible." See [../gin/](../gin/).
