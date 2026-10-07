# React 04 — Data Fetching with TanStack Query (server state)

## Why it matters
"How do you call the API and handle loading, errors, caching, retries?" is asked in every React interview that mentions REST APIs. TanStack Query (React Query) is the de facto answer.

## Why not just `useEffect` + `fetch`?
You'd re-implement: caching, de-duplication, background refetch, retries, race-condition handling, pagination, optimistic updates, request cancellation. A server-state library gives these. Server state is *asynchronous, shared, and can become stale*, which is different from UI state.

> **Clean-architecture note:** TanStack Query belongs to the **presentation** layer. `queryFn`/`mutationFn` should call **use cases** obtained from context (see [08](08-clean-architecture-frontend.md)), not `fetch` directly; DTO parsing and the HTTP client live in `infrastructure/`.

## Setup

```bash
npm i @tanstack/react-query @tanstack/react-query-devtools
```

```tsx
// main.tsx
const queryClient = new QueryClient({
  defaultOptions: { queries: { staleTime: 30_000, retry: 1, refetchOnWindowFocus: true } },
});

createRoot(document.getElementById("root")!).render(
  <QueryClientProvider client={queryClient}>
    <App />
    <ReactQueryDevtools initialIsOpen={false} />
  </QueryClientProvider>
);
```

## The API client (typed, with auth)

```ts
// src/api/client.ts
import { authHeader } from "../auth";

export class ApiError extends Error {
  constructor(public status: number, public body: unknown) { super(`HTTP ${status}`); }
}

export async function api<T>(path: string, init: RequestInit = {}, signal?: AbortSignal): Promise<T> {
  const res = await fetch(`${import.meta.env.VITE_API_URL}${path}`, {
    ...init,
    signal,
    headers: { "Content-Type": "application/json", ...(await authHeader()), ...init.headers },
  });
  if (!res.ok) throw new ApiError(res.status, await res.json().catch(() => null));
  return res.status === 204 ? (undefined as T) : res.json();
}
```

## Queries

```tsx
// src/features/expenses/queries.ts
export const expenseKeys = {
  all: ["expenses"] as const,
  list: (filters: ExpenseFilters) => [...expenseKeys.all, "list", filters] as const,
  detail: (id: string) => [...expenseKeys.all, "detail", id] as const,
};

export function useExpenses(filters: ExpenseFilters) {
  return useQuery({
    queryKey: expenseKeys.list(filters),                     // cache key: change filters → new cache entry
    queryFn: ({ signal }) => api<Page<Expense>>(`/expenses?${toQuery(filters)}`, {}, signal),
    placeholderData: keepPreviousData,                       // no flicker when paging
  });
}
```

```tsx
function ExpensesPage() {
  const [filters, setFilters] = useState<ExpenseFilters>({ month: "2026-10" });
  const { data, isPending, isError, error, isFetching } = useExpenses(filters);

  if (isPending) return <Spinner />;
  if (isError) return <ErrorMessage error={error} />;
  return <ExpenseList items={data.items} refreshing={isFetching} />;
}
```

Concepts:
- **`staleTime`**: how long data counts as fresh (no refetch). **`gcTime`** (formerly cacheTime): how long unused data stays in cache.
- Query keys are *dependency arrays*: include every variable the query function uses.
- `isPending` (no data yet) vs `isFetching` (any request in flight, including background refetch).
- `enabled: !!userId` for dependent queries.

## Mutations with cache updates

```tsx
export function useCreateExpense() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (input: CreateExpense) => api<Expense>("/expenses", { method: "POST", body: JSON.stringify(input) }),
    onSuccess: () => qc.invalidateQueries({ queryKey: expenseKeys.all }),   // refetch lists + reports
  });
}
```

### Optimistic delete with rollback

```tsx
export function useDeleteExpense() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (id: string) => api<void>(`/expenses/${id}`, { method: "DELETE" }),
    onMutate: async (id) => {
      await qc.cancelQueries({ queryKey: expenseKeys.all });
      const snapshots = qc.getQueriesData<Page<Expense>>({ queryKey: expenseKeys.all });
      qc.setQueriesData<Page<Expense>>({ queryKey: expenseKeys.all }, (old) =>
        old ? { ...old, items: old.items.filter((e) => e.id !== id) } : old);
      return { snapshots };
    },
    onError: (_err, _id, ctx) => ctx?.snapshots.forEach(([key, data]) => qc.setQueryData(key, data)),
    onSettled: () => qc.invalidateQueries({ queryKey: expenseKeys.all }),
  });
}
```

## Infinite scroll / cursor pagination (matches the API's `cursor`)

```tsx
const q = useInfiniteQuery({
  queryKey: ["expenses", "infinite", filters],
  queryFn: ({ pageParam, signal }) => api<Page<Expense>>(`/expenses?cursor=${pageParam ?? ""}`, {}, signal),
  initialPageParam: null as string | null,
  getNextPageParam: (last) => last.nextCursor,
});
// q.data.pages.flatMap(p => p.items), q.fetchNextPage(), q.hasNextPage
```

## Auth-aware behavior
Global handling of `401`: in the `QueryCache`'s `onError` redirect to sign-in (or refresh the token once); don't retry 4xx:

```ts
new QueryClient({
  queryCache: new QueryCache({ onError: (e) => { if (e instanceof ApiError && e.status === 401) signOut(); } }),
  defaultOptions: { queries: { retry: (n, e) => !(e instanceof ApiError && e.status < 500) && n < 2 } },
});
```

## Forms + validation (React Hook Form + Zod)

```tsx
const schema = z.object({
  amount: z.coerce.number().positive(),
  category: z.enum(CATEGORIES),
  description: z.string().max(200).optional(),
});
type FormValues = z.infer<typeof schema>;

function ExpenseForm() {
  const { register, handleSubmit, formState: { errors, isSubmitting }, reset } =
    useForm<FormValues>({ resolver: zodResolver(schema) });
  const create = useCreateExpense();

  return (
    <form onSubmit={handleSubmit(async (v) => {
      await create.mutateAsync({ amountCents: Math.round(v.amount * 100), currency: "USD", category: v.category, description: v.description });
      reset();
    })}>
      <input type="number" step="0.01" {...register("amount")} aria-invalid={!!errors.amount} />
      {errors.amount && <span role="alert">{errors.amount.message}</span>}
      <select {...register("category")}>{CATEGORIES.map(c => <option key={c}>{c}</option>)}</select>
      <button disabled={isSubmitting}>Save</button>
      {create.isError && <p role="alert">Could not save. Try again.</p>}
    </form>
  );
}
```
Validate on the client for UX **and** on the server for security. The same Zod schema can be shared or derived from OpenAPI.

## React 19 notes (be aware)
`use(promise)`, `<form action={fn}>`, `useActionState`, `useOptimistic`, and Server Components (frameworks such as Next.js). For a SPA calling a separate API, TanStack Query remains the pragmatic choice. Say that you know where React is heading without over-claiming.

## Exercise
Wire the list, create, optimistic delete and cursor pagination against the FastAPI or Gin backend; add MSW (Mock Service Worker) for tests and Storybook-free local dev.

## Interview Q&A
- **How do you avoid the stale response race condition?** Query keys + `AbortSignal`; the library ignores out-of-date results.
- **`staleTime` vs `gcTime`?** Freshness window vs retention of inactive cache entries.
- **How do you do optimistic updates safely?** Snapshot → apply → rollback on error → invalidate on settle.
- **Where does auth token refresh live?** In the API client (or a single refresh promise shared across concurrent 401s), not in components.
- **Client state vs server state?** Different lifecycle: server state is remote, async, cached and can be out of date.
