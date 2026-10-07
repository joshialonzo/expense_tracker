# TypeScript 06 — Interview Questions and Live-Coding Drills

> Also practice the SOLID/DI questions in [07](07-solid-in-typescript.md).

## Concept questions
1. `any` vs `unknown` vs `never`?
2. `interface` vs `type` — when each?
3. What is structural typing? Excess property checks?
4. What does `strict` enable? Why `noUncheckedIndexedAccess`?
5. Explain generics with a constraint and a default.
6. `keyof`, `typeof`, indexed access `T[K]` — give an example combining them.
7. Discriminated unions and exhaustive checking.
8. Type guards vs assertion functions vs `as`.
9. Are types available at runtime? How do you validate API data?
10. `readonly` vs `as const` vs `Object.freeze`?
11. Enums vs union literals?
12. How does TS handle `this`, function overloads, and `unique symbol`? (overloads: multiple signatures + one implementation)
13. Declaration merging and module augmentation?
14. Covariance/contravariance with `strictFunctionTypes`?
15. How do bundlers and `tsc` split responsibilities?

## Live-coding drills (do each in ≤10 minutes)

### 1. Implement utility types
```ts
type MyPick<T, K extends keyof T> = { [P in K]: T[P] };
type MyExclude<T, U> = T extends U ? never : T;
type MyOmit<T, K extends keyof any> = MyPick<T, MyExclude<keyof T, K>>;
type DeepReadonly<T> = T extends (...a: any[]) => any ? T
  : T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> } : T;
```

### 2. Typed `debounce`
```ts
function debounce<A extends unknown[]>(fn: (...args: A) => void, ms: number) {
  let t: ReturnType<typeof setTimeout> | undefined;
  return (...args: A) => { clearTimeout(t); t = setTimeout(() => fn(...args), ms); };
}
```
Note `ReturnType<typeof setTimeout>` works in both browser and Node.

### 3. Typed event emitter
```ts
type Events = { "expense:created": Expense; "expense:deleted": { id: string } };

class Emitter<E extends Record<string, unknown>> {
  private handlers: { [K in keyof E]?: Array<(p: E[K]) => void> } = {};
  on<K extends keyof E>(k: K, h: (p: E[K]) => void) { (this.handlers[k] ??= []).push(h); }
  emit<K extends keyof E>(k: K, p: E[K]) { this.handlers[k]?.forEach(h => h(p)); }
}
const bus = new Emitter<Events>();
bus.on("expense:deleted", ({ id }) => console.log(id));
```

### 4. Fix this code
```ts
function total(items: any) { return items.reduce((s, i) => s + i.amount, 0); }
```
Expected: type `items` as `readonly { amountCents: number }[]`, return `number`, and note `any` hid the typo risk (`amount` vs `amountCents`).

### 5. Retry with generics
```ts
async function retry<T>(fn: () => Promise<T>, attempts = 3, delayMs = 200): Promise<T> {
  let last: unknown;
  for (let i = 0; i < attempts; i++) {
    try { return await fn(); }
    catch (e) { last = e; await new Promise(r => setTimeout(r, delayMs * 2 ** i)); }
  }
  throw last;
}
```
Follow-ups: jitter, only retry on 5xx/network errors, honor `AbortSignal`.

### 6. Safe `get` by path
Write `get<T, K extends keyof T>(o: T, k: K): T[K]`, then extend to nested paths using template literal types (stretch).

## JavaScript fundamentals that often accompany TS questions
- Closures, `this` binding (arrow functions don't bind), prototypes/classes.
- `==` vs `===`, truthy/falsy, `NaN`, floating point (`0.1+0.2`) → money as integer cents.
- Shallow vs deep copy (`structuredClone`), immutability with spread.
- `map/filter/reduce`, `Object.entries`, optional chaining.
- Event loop, microtasks vs macrotasks ([04](04-async-modules-and-tooling.md)).
- Debounce vs throttle.

## How to talk about TS on your résumé
"I enable `strict`, derive types from schemas (Zod/OpenAPI) so runtime and compile time can't drift, model state with discriminated unions, and gate CI with `tsc --noEmit`, ESLint and tests."
