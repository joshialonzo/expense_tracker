# TypeScript 03 — Narrowing, Discriminated Unions and Type-Safe State

## Why it matters
Modeling states with discriminated unions is the "senior" signal: it makes impossible states unrepresentable, and it is how you'd type reducers, API results and MCP messages.

## Narrowing techniques

```ts
function describe(x: string | number | Date | null | undefined) {
  if (x == null) return "nothing";            // null and undefined
  if (typeof x === "string") return x.toUpperCase();
  if (typeof x === "number") return x.toFixed(2);
  return x.toISOString();                     // Date by elimination
}

// instanceof, in
function msg(e: unknown) {
  if (e instanceof Error) return e.message;
  if (typeof e === "object" && e !== null && "message" in e) return String((e as { message: unknown }).message);
  return String(e);
}

// custom type guard
function isExpense(v: unknown): v is Expense {
  return typeof v === "object" && v !== null && "amountCents" in v && "currency" in v;
}

// assertion function
function assertDefined<T>(v: T | undefined, name: string): asserts v is T {
  if (v === undefined) throw new Error(`${name} is required`);
}
```
(Type guards are only as correct as you write them. Prefer a Zod `safeParse`.)

## Discriminated unions

A shared literal property (the *discriminant*) lets TS narrow the whole object.

```ts
type RemoteData<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: string };

function render(state: RemoteData<Expense[]>) {
  switch (state.status) {
    case "idle":    return "Click load";
    case "loading": return "Loading…";
    case "success": return `${state.data.length} expenses`;   // data exists only here
    case "error":   return `Failed: ${state.error}`;          // error exists only here
    default: return assertNever(state);
  }
}

function assertNever(x: never): never {
  throw new Error(`Unhandled case: ${JSON.stringify(x)}`);
}
```
Add a new status and TS flags every `switch` missing it, thanks to the `never` exhaustiveness check.

Contrast with the anti-pattern: `{ loading: boolean; data?: T; error?: string }` allows `loading: true` with `error` set, an impossible state.

## Typed reducer (the React `useReducer` pattern)

```ts
type State = { items: Expense[]; filter: Category | "all" };

type Action =
  | { type: "added"; expense: Expense }
  | { type: "removed"; id: string }
  | { type: "filtered"; category: Category | "all" };

export function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "added":    return { ...state, items: [action.expense, ...state.items] };
    case "removed":  return { ...state, items: state.items.filter(e => e.id !== action.id) };
    case "filtered": return { ...state, filter: action.category };
  }
}
```
With a full `switch` over a union return type, TS errors if a case is missing; no `default` needed.

## `Result` and error handling

`catch (e)` is `unknown` under `strict` (`useUnknownInCatchVariables`). Narrow it. Return `Result<T>` from functions when failure is expected (validation), and throw for truly exceptional cases.

## Type-level safety for API shapes

```ts
type ApiErrorBody =
  | { code: "validation_error"; fields: Record<string, string> }
  | { code: "not_found" }
  | { code: "unauthorized" };

function toMessage(e: ApiErrorBody): string {
  switch (e.code) {
    case "validation_error": return Object.values(e.fields).join(", ");
    case "not_found": return "Not found";
    case "unauthorized": return "Please sign in";
  }
}
```

## Branded types (nice extra)
Stop mixing up IDs and units:

```ts
type Brand<T, B extends string> = T & { readonly __brand: B };
type UserId = Brand<string, "UserId">;
type ExpenseId = Brand<string, "ExpenseId">;
type Cents = Brand<number, "Cents">;

const asCents = (n: number) => { if (!Number.isInteger(n)) throw new Error("cents must be int"); return n as Cents; };
declare function deleteExpense(id: ExpenseId): void;
// deleteExpense("abc" as UserId)  // ✘ compile error
```

## Exercise
1. Replace `{loading, data, error}` booleans in a component with `RemoteData<T>` and render with an exhaustive switch.
2. Add a new action `"edited"` to the reducer and observe what the compiler complains about.
3. Model a payment-method union (`card | bank | cash`) with different fields and write `describePayment`.

## Interview Q&A
- **What is a discriminated union and why use it?** Union of object types sharing a literal tag; enables exhaustive, safe narrowing and impossible-state elimination.
- **How do you do an exhaustive check?** Assign the remaining value to `never` in `default`.
- **`unknown` in `catch`—how do you handle it?** Narrow with `instanceof Error`, else stringify.
- **Type guard vs assertion function?** Boolean-returning `x is T` vs throws-or-narrows `asserts x is T`.
