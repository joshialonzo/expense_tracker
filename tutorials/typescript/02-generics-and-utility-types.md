# TypeScript 02 — Generics, Utility Types, Mapped and Conditional Types

## Why it matters
Generics are the favorite TS interview topic: "write a typed `fetch` wrapper", "type this hook", "implement `Pick`".

## Generics

A generic is a type parameter, i.e. a function for types.

```ts
function first<T>(items: readonly T[]): T | undefined {
  return items[0];
}
first([1, 2]);        // number | undefined
first(["a"]);         // string | undefined

// constraint: T must have an id
function indexById<T extends { id: string }>(items: T[]): Map<string, T> {
  return new Map(items.map(i => [i.id, i]));
}

// two params, relating them
function pluck<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
pluck({ id: "1", amountCents: 5 }, "amountCents");  // number
```

Default type params: `interface ApiResponse<T = unknown> { data: T; error?: string }`.

### Typed API client (very likely live-coding)

```ts
type Json = string | number | boolean | null | Json[] | { [k: string]: Json };

export class ApiError extends Error {
  constructor(public status: number, message: string, public body?: unknown) {
    super(message);
  }
}

export async function request<T>(
  path: string,
  init?: Omit<RequestInit, "body"> & { body?: Json },
  parse: (raw: unknown) => T = (raw) => raw as T,     // plug in a Zod schema's parse
): Promise<T> {
  const res = await fetch(`${import.meta.env.VITE_API_URL}${path}`, {
    ...init,
    headers: { "Content-Type": "application/json", ...init?.headers },
    body: init?.body === undefined ? undefined : JSON.stringify(init.body),
  });
  if (!res.ok) throw new ApiError(res.status, res.statusText, await res.json().catch(() => undefined));
  if (res.status === 204) return undefined as T;
  return parse(await res.json());
}

// usage
const expenses = await request("/expenses", undefined, (raw) => z.array(ExpenseSchema).parse(raw));
```

### Generic constraints with `keyof` — group by

```ts
function groupBy<T, K extends PropertyKey>(items: T[], key: (item: T) => K): Record<K, T[]> {
  return items.reduce((acc, item) => {
    (acc[key(item)] ??= []).push(item);
    return acc;
  }, {} as Record<K, T[]>);
}
const byCategory = groupBy(expenses, e => e.category);
```

## Built-in utility types (know these by heart)

| Utility | Meaning | Example |
|---|---|---|
| `Partial<T>` | all props optional | `PATCH` body |
| `Required<T>` | all props required | |
| `Readonly<T>` | all readonly | |
| `Pick<T, K>` | keep props K | `Pick<Expense, "id"\|"amountCents">` |
| `Omit<T, K>` | drop props K | `Omit<Expense, "id"\|"createdAt">` = create DTO |
| `Record<K, V>` | object from keys to V | `Record<Category, number>` |
| `Exclude<U, X>` / `Extract<U, X>` | filter a union | |
| `NonNullable<T>` | remove null/undefined | |
| `ReturnType<F>` / `Parameters<F>` | derive from function | |
| `Awaited<T>` | unwrap Promise | `Awaited<ReturnType<typeof getExpense>>` |

```ts
type CreateExpense = Omit<Expense, "id" | "createdAt">;
type UpdateExpense = Partial<CreateExpense>;
type TotalsByCategory = Record<Category, number>;
```

## Mapped types
Implement `Partial` yourself:

```ts
type MyPartial<T> = { [K in keyof T]?: T[K] };
type MyReadonly<T> = { readonly [K in keyof T]: T[K] };
type Nullable<T> = { [K in keyof T]: T[K] | null };

// key remapping
type Getters<T> = { [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K] };
type G = Getters<{ id: string; amountCents: number }>;
// { getId: () => string; getAmountCents: () => number }
```

## Conditional types and `infer`

```ts
type IsArray<T> = T extends unknown[] ? true : false;

type ElementOf<T> = T extends (infer U)[] ? U : never;
type E = ElementOf<Expense[]>;          // Expense

// re-implement ReturnType
type MyReturnType<F> = F extends (...args: any[]) => infer R ? R : never;

// Distributive over unions
type ToArray<T> = T extends unknown ? T[] : never;
type X = ToArray<string | number>;      // string[] | number[]
```

## Template literal types

```ts
type Route = `/expenses` | `/expenses/${string}` | `/reports/summary`;
type EventName = `${"expense"}:${"created" | "deleted"}`;   // "expense:created" | "expense:deleted"
```

## Type-safe event/result patterns

```ts
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

async function safe<T>(p: Promise<T>): Promise<Result<T>> {
  try { return { ok: true, value: await p }; }
  catch (e) { return { ok: false, error: e instanceof Error ? e : new Error(String(e)) }; }
}
```

## Exercise
Implement from scratch (no lib utilities): `MyPick`, `MyOmit`, `DeepReadonly`, `Awaited`-like `Unwrap<T>`, and `Merge<A, B>`. Then type `useFetch<T>` for [react/04](../react/04-data-fetching-tanstack-query.md).

## Interview Q&A
- **What problem do generics solve?** Reuse with type relationships preserved (input type determines output type) vs `any`.
- **`extends` in a generic vs conditional type?** Constraint ("T must be a subtype of") vs condition (a ternary on types).
- **What does `keyof typeof obj` give you?** Union of the object's key names.
- **Implement `Omit`.** `type MyOmit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>`.
- **Variance / why `string[]` is assignable to `readonly string[]` but not the reverse?** Mutability: writing through the wider reference could break the narrower one.
