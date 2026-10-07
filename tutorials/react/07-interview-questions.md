# React 07 — Interview Questions and Live-Coding Drills

## Conceptual (answer in ≤ 60 s each)
1. How does React render and update the DOM? (render phase → reconcile → commit)
2. Why are keys important? What goes wrong with index keys?
3. Props vs state; lifting state up; controlled vs uncontrolled.
4. Explain `useEffect` dependencies and cleanup. Why does it run twice in dev?
5. `useMemo`/`useCallback`/`React.memo`: when do they help and when do they hurt?
6. Context vs Redux/Zustand. Re-render implications.
7. How would you fetch data? What does TanStack Query give you?
8. Explain stale closures with an example.
9. How do you handle errors (error boundaries, query errors, form errors)?
10. Code splitting and Suspense.
11. What changed in React 18/19? (concurrent rendering, automatic batching, `useTransition`, `use`, Actions, Server Components)
12. How would you secure a React app? (XSS, token storage, CSP, dependency audit, no secrets in bundle)
13. Accessibility checklist.
14. SSR vs CSR vs SSG; when would you choose Next.js?
15. How do you test React components?

## Live-coding drills

### Drill 1 — Debounced search with abort (15 min)
Build `<SearchExpenses>` that calls `GET /expenses?q=` after 300 ms of idle, cancels in-flight requests, shows loading/empty/error. Expected: `useDebounced`, `AbortController` or TanStack Query with `signal`, discriminated union state.

### Drill 2 — Todo-style CRUD with reducer (15 min)
Add/remove/toggle with `useReducer`; unit-test the reducer.

### Drill 3 — Fix the bug
```tsx
function Timer() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    setInterval(() => setCount(count + 1), 1000);
  }, []);
  return <p>{count}</p>;
}
```
Problems: stale closure (`count` stays 0 → stuck at 1), no cleanup (leaks, doubles under StrictMode). Fix:
```tsx
useEffect(() => {
  const id = setInterval(() => setCount(c => c + 1), 1000);
  return () => clearInterval(id);
}, []);
```

### Drill 4 — Fix the performance
```tsx
function Page() {
  const [text, setText] = useState("");
  const rows = expensiveRows();            // 10k rows recomputed on every keystroke
  return <><input value={text} onChange={e => setText(e.target.value)} /><Table rows={rows} /></>;
}
```
Options: move `rows` out/`useMemo`, move the input into its own component (state down), memoize `Table`, virtualize.

### Drill 5 — Build `useFetch<T>` generic hook
Return `RemoteData<T>`, handle abort, re-fetch on URL change, and explain why you'd prefer TanStack Query in production.

### Drill 6 — Build an accessible modal / dropdown / tabs
Focus trap, `Escape` closes, `aria-modal`, return focus. (Or say you'd use Radix/Headless UI and explain why; knowing what they handle is the point.)

### Drill 7 — Infinite scroll with `IntersectionObserver`
Sentinel element + `useInfiniteQuery.fetchNextPage`.

## Machine-coding style prompt for this repo
"Build the expense dashboard: list with filters synced to URL, add expense form with validation, optimistic delete, monthly total by category chart, loading/error states, tests." Time-box: 45 minutes. Review against [05](05-routing-auth-and-structure.md) and [06](06-performance-and-testing.md).

## Talking points that signal seniority
- "I keep server state in a query cache and URL state in the URL, which leaves very little true global state."
- "I model UI states as discriminated unions so impossible states can't compile."
- "I measure before optimizing and I test behavior through the DOM with MSW."
- "Client guards are UX; authorization is enforced server-side."
- "I avoid effects for derived data."

## React vs Angular (they list Angular as nice-to-have)
See [../angular/05-react-vs-angular-and-interview.md](../angular/05-react-vs-angular-and-interview.md).
