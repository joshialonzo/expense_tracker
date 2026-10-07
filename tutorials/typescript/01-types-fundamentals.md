# TypeScript 01 — Type System Fundamentals

## Why it matters
TypeScript is a "must have" topic and every other stack here (React, Angular, MCP TS SDK, CDK) is written in it. Expect live-coding with types.

## Mental model
- TS is a **compile-time, structurally typed** layer over JavaScript. Types are erased at runtime, so they don't validate incoming data (see "runtime validation" below).
- **Structural typing**: compatibility is about shape, not name.

```ts
interface Expense {
  id: string;
  amountCents: number;
  currency: "USD" | "EUR" | "MXN";
  category: string;
  description?: string;     // optional
  readonly createdAt: Date; // cannot be reassigned
}

const e = { id: "1", amountCents: 1250, currency: "USD", category: "food", createdAt: new Date(), extra: 1 };
const typed: Expense = e;   // OK: extra props are allowed when not a fresh object literal
```

## The basics you must know cold

```ts
// primitives
let n: number = 1; let s: string = "a"; let b: boolean = true;
let big: bigint = 10n; let sym: symbol = Symbol("k");

// arrays/tuples
const ids: string[] = ["a", "b"];
const pair: [string, number] = ["food", 1250];

// unions & literals
type Currency = "USD" | "EUR";
type Id = string | number;

// any vs unknown vs never
let a: any;            // disables checking — avoid
let u: unknown;        // must narrow before use — prefer
function fail(msg: string): never { throw new Error(msg); }  // never returns

// functions
function total(expenses: Expense[], currency?: Currency): number {
  return expenses.filter(e => !currency || e.currency === currency)
                 .reduce((sum, e) => sum + e.amountCents, 0);
}
const fmt = (cents: number): string => (cents / 100).toFixed(2);
```

### `interface` vs `type`

| | `interface` | `type` |
|---|---|---|
| Object shapes | ✔ | ✔ |
| Extend | `extends`, declaration merging | intersection `&` |
| Unions, tuples, mapped/conditional types | ✘ | ✔ |
| Classes `implements` | ✔ | ✔ (object types) |

Rule of thumb: `interface` for public object contracts (and library augmentation), `type` for unions and computed types. Be consistent; either answer is fine if you can explain it.

### `enum` vs union of literals
Prefer string-literal unions (`type Category = "food" | "rent"`) or `as const` objects: no runtime code, better tree shaking, no enum quirks.

```ts
export const CATEGORIES = ["food", "transport", "housing", "fun"] as const;
export type Category = (typeof CATEGORIES)[number];   // "food" | "transport" | ...
```
This single source of truth gives a runtime array (for a `<select>`) and a type.

### `as`, `!`, and `satisfies`
- `value as Foo` asserts; it does **not** convert or check. Overuse hides bugs.
- `x!` non-null assertion: same risk.
- `satisfies` checks a value against a type *without widening it*:

```ts
const colors = {
  food: "#f97316",
  transport: "#3b82f6",
} satisfies Record<string, `#${string}`>;
colors.food;   // still typed as string literal-ish keys, autocomplete works
```

### Strict mode
Always `"strict": true` (enables `strictNullChecks`, `noImplicitAny`, etc.). Add `noUncheckedIndexedAccess` to make `arr[i]` possibly `undefined`.

### `null` / `undefined`
With `strictNullChecks`, `string` excludes `null`. Use optional chaining `a?.b`, nullish coalescing `a ?? default` (not `||`, which treats `0` and `""` as falsy — a bug for `amountCents: 0`).

## Runtime validation: types don't exist at runtime
`fetch().then(r => r.json())` returns `any`; annotating it is a lie. Validate at the boundary with **Zod** and infer the type:

```ts
import { z } from "zod";

export const ExpenseSchema = z.object({
  id: z.string().uuid(),
  amountCents: z.number().int().positive(),
  currency: z.enum(["USD", "EUR", "MXN"]),
  category: z.string().min(1),
  description: z.string().optional(),
  createdAt: z.coerce.date(),
});
export type Expense = z.infer<typeof ExpenseSchema>;

export async function getExpense(id: string): Promise<Expense> {
  const res = await fetch(`/api/expenses/${id}`);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return ExpenseSchema.parse(await res.json());   // throws with details if the API drifts
}
```

## Exercise
1. Create `types.ts` with `Expense`, `Category` (from an `as const` array) and a `Money` helper `formatMoney(cents, currency)` using `Intl.NumberFormat`.
2. Turn on `strict` and `noUncheckedIndexedAccess`; fix all errors without using `any`, `as` or `!`.
3. Write the Zod schema and prove it rejects `{ amountCents: -5 }`.

## Interview Q&A
- **`any` vs `unknown`?** `any` opts out of checking; `unknown` is the type-safe top type, requiring narrowing.
- **Are TS types available at runtime?** No (erased); use schemas (Zod) or type guards for validation.
- **`interface` vs `type`?** See table; declaration merging and unions are the key differences.
- **What is structural typing?** Compatibility by shape, so duck typing with compile-time checks.
- **`==` vs `??`/`||`?** Use `??` for null/undefined defaults.
- **Why `readonly`/`as const`?** Immutability signals and narrower literal types.
