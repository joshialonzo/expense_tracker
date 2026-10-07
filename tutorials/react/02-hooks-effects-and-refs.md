# React 02 — Hooks: useEffect, useRef, useMemo, useCallback and the Rules

## Why it matters
Hooks questions are guaranteed: "explain `useEffect` dependencies", "why is my effect running twice", "when to use `useMemo`".

## Rules of Hooks
1. Call hooks only at the **top level** (not in conditions/loops/nested functions).
2. Call only from components or custom hooks.
Reason: React identifies each hook by call order per component. Enable `eslint-plugin-react-hooks` (`rules-of-hooks`, `exhaustive-deps`).

## useState recap
- Lazy initializer for expensive init: `useState(() => parse(localStorage.getItem("x")))`.
- Objects: replace, don't mutate: `setForm(f => ({ ...f, amount: 5 }))`.

## useEffect: synchronize with something *outside* React

Effects run **after render/commit**. Use them for: subscriptions, timers, non-React widgets, network sync, document title. **Do not** use them for derived data or event-driven logic ("when user clicks Save, POST" belongs in the handler).

```tsx
useEffect(() => {
  // setup
  const id = setInterval(() => setTick(t => t + 1), 1000);
  return () => clearInterval(id);        // cleanup: before re-run and on unmount
}, []);                                   // dependency array
```

Dependency array semantics:
- omitted → runs after **every** render
- `[]` → once after mount (and cleanup on unmount)
- `[a, b]` → when `a` or `b` change (compared with `Object.is`)

**Strict Mode (dev)** mounts → unmounts → mounts to expose missing cleanups. That's why effects "run twice" in development. The fix is a correct cleanup, not removing StrictMode.

### Fetching in an effect (the right way, if not using a library)

```tsx
function useExpenses(month: string) {
  const [state, setState] = useState<RemoteData<Expense[]>>({ status: "loading" });

  useEffect(() => {
    const ctrl = new AbortController();
    setState({ status: "loading" });
    fetch(`/api/expenses?month=${encodeURIComponent(month)}`, { signal: ctrl.signal })
      .then((r) => { if (!r.ok) throw new Error(`HTTP ${r.status}`); return r.json(); })
      .then((data: Expense[]) => setState({ status: "success", data }))
      .catch((e) => { if (e.name !== "AbortError") setState({ status: "error", error: String(e) }); });
    return () => ctrl.abort();           // avoids race conditions & updates after unmount
  }, [month]);

  return state;
}
```
Race condition: if `month` changes quickly, an old slow response must not overwrite a newer one. Abort/ignore flag solves it. In real apps, use TanStack Query ([04](04-data-fetching-tanstack-query.md)) instead of hand-rolling.

### "You might not need an effect"
- Derived value → compute in render (`useMemo` if expensive).
- Reset state when a prop changes → give the component a `key`.
- Notify parent after state change → call parent's callback in the same event handler.
- Chains of effects setting state → combine into one handler/reducer.

## useRef
Mutable box whose changes **don't** re-render. Two uses: DOM node access and persisting values across renders (timer IDs, previous value, latest callback).

```tsx
function useDebounced<T>(value: T, ms = 300): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const t = setTimeout(() => setDebounced(value), ms);
    return () => clearTimeout(t);
  }, [value, ms]);
  return debounced;
}

function usePrevious<T>(value: T) {
  const ref = useRef<T | undefined>(undefined);
  useEffect(() => { ref.current = value; });
  return ref.current;
}
```
(Don't read/write `ref.current` during render for logic that affects output.)

## useMemo / useCallback / React.memo

- `useMemo(() => expensive(a, b), [a, b])` caches a computed value.
- `useCallback(fn, deps)` caches a function identity (≈ `useMemo(() => fn, deps)`).
- `React.memo(Component)` skips re-render when props are shallow-equal.

They matter only when: (1) the calculation is genuinely expensive, or (2) you pass a value to a **memoized** child/effect dependency and a new identity each render would defeat it. Otherwise they add cost and noise. **Measure with the React DevTools Profiler first.** (React Compiler, once enabled, memoizes automatically so you write less of this by hand.)

```tsx
const Row = React.memo(function Row({ expense, onDelete }: RowProps) { /* … */ });

function List({ expenses }: { expenses: Expense[] }) {
  const onDelete = useCallback((id: string) => api.delete(id), []);   // stable identity for memoized Row
  const total = useMemo(() => expenses.reduce((s, e) => s + e.amountCents, 0), [expenses]);
  return <>{expenses.map(e => <Row key={e.id} expense={e} onDelete={onDelete} />)}<p>{total}</p></>;
}
```

## useLayoutEffect, useId, useTransition, useDeferredValue (know they exist)
- `useLayoutEffect`: synchronous after DOM mutation, before paint (measure layout). Rare.
- `useId`: stable unique IDs for label/input pairs (SSR-safe).
- `useTransition` / `useDeferredValue`: mark updates as non-urgent so typing stays responsive while a heavy list re-filters.

```tsx
const [q, setQ] = useState("");
const deferredQ = useDeferredValue(q);
const results = useMemo(() => search(expenses, deferredQ), [expenses, deferredQ]);
```

## Stale closures
Effects/handlers capture the values from the render they were created in. A `setInterval(() => console.log(count), 1000)` created with `[]` logs the initial `count` forever. Fix: functional updates, add dependencies, or a ref holding the latest value.

## Exercise
1. Build `useDebounced` and a search box that queries the API only after 300 ms idle, with abort on change.
2. Write an effect with a deliberate missing cleanup, observe duplicate listeners under StrictMode, fix it.
3. Profile a 5,000-row list; make it fast with `React.memo` + stable callbacks (then try virtualization with `@tanstack/react-virtual`).

## Interview Q&A
- **`useEffect` vs `useLayoutEffect`?** After paint (async) vs before paint (sync); the latter blocks painting.
- **What happens with `[]` and a prop used inside?** Stale closure; the lint rule flags it.
- **Why does my effect run twice?** StrictMode dev double-invoke; ensure idempotent setup/cleanup.
- **When not to use `useMemo`?** Cheap computations, or when the dependency changes every render anyway.
- **Why can't hooks be conditional?** Call-order-based slots.
- **`useRef` vs `useState`?** Ref doesn't trigger re-render.
