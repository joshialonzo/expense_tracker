# React 08 — Clean Architecture and SOLID on the Front-End

Principles: [../fullstack/08](../fullstack/08-solid-and-clean-architecture.md) and [../typescript/07](../typescript/07-solid-in-typescript.md). This is the **reference structure for `web-react/`**. The Angular twin is [../angular/06](../angular/06-clean-architecture-frontend.md): the first three layers are **identical TypeScript**, only presentation differs, which is exactly the point.

## Why bother on the front-end?
Typical React smell: components that `fetch`, validate, transform DTOs, and hold business rules inside `useEffect`. That code can't be reused by Angular, can't be tested without a DOM + network, and breaks whenever the API shape changes. Move everything that isn't "draw pixels and handle events" out of components.

> **Implemented and tested in [`web-react/`](../../web-react)** (71 tests, including an architecture test that fails if a layer imports something it must not). Note: current Vite templates enable `erasableSyntaxOnly`, which forbids TypeScript parameter properties (`constructor(private x: T)`), so classes declare fields explicitly. The architecture rules there are a Vitest test rather than ESLint config, so they run with the normal test command.

## Layers and the dependency rule

```
presentation  (React components, hooks, pages, TanStack Query)   ──►  application
infrastructure (HttpExpenseGateway, auth token, Zod DTOs)         ──►  application   (implements ports)
application   (use cases + ports)                                 ──►  domain
domain        (entities, value objects, pure rules)               ──►  nothing
app/composition.ts wires everything (the only file that sees all layers)
```

```
web-react/src/
  domain/
    money.ts  expense.ts  category.ts  summary.ts  errors.ts  result.ts
  application/
    ports.ts                       # ExpenseGateway (reader/writer), Clock, AssistantGateway
    addExpense.ts  listExpenses.ts  removeExpense.ts  monthlyTotals.ts  askAssistant.ts
    index.ts                       # UseCases type
  infrastructure/
    http/HttpExpenseGateway.ts  http/dto.ts  http/apiClient.ts
    auth/tokenProvider.ts
    system/clock.ts
    fake/FakeExpenseGateway.ts     # also used by Storybook/tests/local dev
  presentation/
    providers/UseCasesProvider.tsx  providers/QueryProvider.tsx
    hooks/useExpenses.ts  hooks/useAddExpense.ts  hooks/useMonthlyTotals.ts
    components/…  pages/…  routes.tsx
  app/composition.ts  main.tsx
```

## Domain (pure TypeScript, no React, no fetch)

```ts
// domain/result.ts
export type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };
export const ok = <T>(value: T): Result<T, never> => ({ ok: true, value });
export const err = <E>(error: E): Result<never, E> => ({ ok: false, error });

// domain/errors.ts
export type ValidationIssue = { field: string; message: string };
export class NotFoundError extends Error { constructor(m = "not found") { super(m); this.name = "NotFoundError"; } }
export class UnauthorizedError extends Error { constructor() { super("unauthorized"); this.name = "UnauthorizedError"; } }
```

```ts
// domain/money.ts
export type Currency = "USD" | "EUR" | "MXN";
export type Money = { readonly cents: number; readonly currency: Currency };

export const money = (cents: number, currency: Currency = "USD"): Money => {
  if (!Number.isInteger(cents) || cents <= 0) throw new RangeError("cents must be a positive integer");
  return { cents, currency };
};
/** parse user text like "12.50" → cents without float math */
export function parseAmount(text: string): number | null {
  const m = /^\s*(\d+)(?:[.,](\d{1,2}))?\s*$/.exec(text);
  return m ? Number(m[1]) * 100 + Number((m[2] ?? "").padEnd(2, "0")) : null;
}
export const formatMoney = (m: Money, locale = "en-US") =>
  new Intl.NumberFormat(locale, { style: "currency", currency: m.currency }).format(m.cents / 100);
```

```ts
// domain/category.ts
export const CATEGORIES = ["food", "transport", "housing", "fun", "other"] as const;
export type Category = (typeof CATEGORIES)[number];
export const isCategory = (v: string): v is Category => (CATEGORIES as readonly string[]).includes(v);

// domain/expense.ts
export type Expense = Readonly<{
  id: string; amount: Money; category: Category; spentOn: string /* YYYY-MM-DD */; description?: string;
}>;
export type NewExpense = Omit<Expense, "id">;

export type ExpenseDraft = { amount: string; category: string; spentOn: string; description?: string };

/** Pure validation → either a valid NewExpense or every issue found (form-friendly). */
export function parseNewExpense(draft: ExpenseDraft, today: string): Result<NewExpense, ValidationIssue[]> {
  const issues: ValidationIssue[] = [];
  const cents = parseAmount(draft.amount);
  if (cents === null || cents <= 0) issues.push({ field: "amount", message: "Enter an amount greater than 0" });
  if (!isCategory(draft.category)) issues.push({ field: "category", message: "Choose a category" });
  if (!/^\d{4}-\d{2}-\d{2}$/.test(draft.spentOn)) issues.push({ field: "spentOn", message: "Invalid date" });
  else if (draft.spentOn > today) issues.push({ field: "spentOn", message: "Date can't be in the future" });
  if ((draft.description ?? "").length > 200) issues.push({ field: "description", message: "Max 200 characters" });
  if (issues.length) return err(issues);
  return ok({ amount: money(cents!), category: draft.category as Category, spentOn: draft.spentOn,
              description: draft.description?.trim() || undefined });
}

// domain/summary.ts
export function totalsByCategory(expenses: readonly Expense[]): Record<Category, number> {
  const out = Object.fromEntries(CATEGORIES.map(c => [c, 0])) as Record<Category, number>;
  for (const e of expenses) out[e.category] += e.amount.cents;
  return out;
}
```
The same validation that the server enforces runs here for UX; the server remains the authority.

## Application: ports + use cases

```ts
// application/ports.ts
import type { Expense, NewExpense } from "../domain/expense";

export type Page<T> = { items: T[]; nextCursor: string | null };
export type ExpenseQuery = { from: string; to: string; category?: string; cursor?: string };

/** ISP: separate reader/writer roles */
export interface ExpenseReader {
  list(q: ExpenseQuery, signal?: AbortSignal): Promise<Page<Expense>>;
}
export interface ExpenseWriter {
  add(e: NewExpense): Promise<Expense>;
  remove(id: string): Promise<void>;
}
export interface Clock { today(): string }
export interface AssistantGateway { ask(message: string): Promise<{ answer: string; toolCalls: { name: string }[] }> }
```

```ts
// application/addExpense.ts
import { parseNewExpense, type Expense, type ExpenseDraft } from "../domain/expense";
import type { Result, ValidationIssue } from "../domain/result";
import type { Clock, ExpenseWriter } from "./ports";

export const makeAddExpense = (deps: { writer: ExpenseWriter; clock: Clock }) =>
  async (draft: ExpenseDraft): Promise<Result<Expense, ValidationIssue[]>> => {
    const parsed = parseNewExpense(draft, deps.clock.today());
    if (!parsed.ok) return parsed;
    return { ok: true, value: await deps.writer.add(parsed.value) };
  };
```
```ts
// application/monthlyTotals.ts
export const makeMonthlyTotals = (deps: { reader: ExpenseReader }) =>
  async (month: string, signal?: AbortSignal) => {
    const items: Expense[] = []; let cursor: string | undefined;
    do {
      const page = await deps.reader.list({ from: `${month}-01`, to: `${month}-31`, cursor }, signal);
      items.push(...page.items); cursor = page.nextCursor ?? undefined;
    } while (cursor);
    return totalsByCategory(items);
  };
```
```ts
// application/index.ts
export type UseCases = {
  addExpense: ReturnType<typeof makeAddExpense>;
  listExpenses: ReturnType<typeof makeListExpenses>;
  removeExpense: ReturnType<typeof makeRemoveExpense>;
  monthlyTotals: ReturnType<typeof makeMonthlyTotals>;
  askAssistant: ReturnType<typeof makeAskAssistant>;
};
```

## Infrastructure (adapters): the only code that knows HTTP and the wire format

```ts
// infrastructure/http/dto.ts  — anti-corruption layer: wire shape ↔ domain
import { z } from "zod";
import { money, type Currency } from "../../domain/money";
import type { Expense } from "../../domain/expense";
import type { Category } from "../../domain/category";

export const ExpenseDto = z.object({
  id: z.string(), amountCents: z.number().int().positive(), currency: z.enum(["USD", "EUR", "MXN"]),
  category: z.enum(["food", "transport", "housing", "fun", "other"]),
  description: z.string().nullish(), date: z.string(),
});
export const PageDto = z.object({ items: z.array(ExpenseDto), nextCursor: z.string().nullable() });

export const toDomain = (d: z.infer<typeof ExpenseDto>): Expense => ({
  id: d.id, amount: money(d.amountCents, d.currency as Currency), category: d.category as Category,
  spentOn: d.date, description: d.description ?? undefined,
});
export const toCreateBody = (e: NewExpense) => ({
  amountCents: e.amount.cents, currency: e.amount.currency, category: e.category, date: e.spentOn, description: e.description,
});
```
```ts
// infrastructure/http/HttpExpenseGateway.ts
import type { ExpenseReader, ExpenseWriter, ExpenseQuery, Page } from "../../application/ports";
import { NotFoundError, UnauthorizedError } from "../../domain/errors";

export class HttpExpenseGateway implements ExpenseReader, ExpenseWriter {
  private readonly api: ApiClient

  constructor(api: ApiClient) {
    this.api = api
  }

  async list(q: ExpenseQuery, signal?: AbortSignal): Promise<Page<Expense>> {
    const params = new URLSearchParams({ from: q.from, to: q.to, ...(q.category && { category: q.category }), ...(q.cursor && { cursor: q.cursor }) });
    const dto = PageDto.parse(await this.api.get(`/expenses?${params}`, signal));
    return { items: dto.items.map(toDomain), nextCursor: dto.nextCursor };
  }
  async add(e: NewExpense) { return toDomain(ExpenseDto.parse(await this.api.post("/expenses", toCreateBody(e)))); }
  async remove(id: string) { await this.api.delete(`/expenses/${id}`); }
}

// apiClient.ts: maps HTTP failures to domain/app errors (401 → UnauthorizedError, 404 → NotFoundError)
```
Backend renames a field? Only `dto.ts` changes.

```ts
// infrastructure/fake/FakeExpenseGateway.ts — in-memory twin (tests, Storybook, offline dev)
export class FakeExpenseGateway implements ExpenseReader, ExpenseWriter {
  private items: Expense[] = []; private n = 0;
  async list(q: ExpenseQuery) { return { items: this.items.filter(e => e.spentOn >= q.from && e.spentOn <= q.to), nextCursor: null }; }
  async add(e: NewExpense) { const x = { ...e, id: `e${++this.n}` }; this.items.unshift(x); return x; }
  async remove(id: string) { this.items = this.items.filter(e => e.id !== id); }
}
```

## Composition root and React integration

```ts
// app/composition.ts
export function buildUseCases(cfg: RuntimeConfig): UseCases {
  const api = new ApiClient(cfg.apiBase, tokenProvider);
  const gateway = new HttpExpenseGateway(api);
  const clock = systemClock;
  return {
    addExpense: makeAddExpense({ writer: gateway, clock }),
    listExpenses: makeListExpenses({ reader: gateway }),
    removeExpense: makeRemoveExpense({ writer: gateway }),
    monthlyTotals: makeMonthlyTotals({ reader: gateway }),
    askAssistant: makeAskAssistant({ gateway: new HttpAssistantGateway(api) }),
  };
}
```
```tsx
// presentation/providers/UseCasesProvider.tsx  — context = injection mechanism for the UI layer
const UseCasesContext = createContext<UseCases | null>(null);
export const UseCasesProvider = ({ useCases, children }: { useCases: UseCases; children: React.ReactNode }) =>
  <UseCasesContext value={useCases}>{children}</UseCasesContext>;     // React 19; use .Provider in 18
export function useUseCases(): UseCases {
  const v = useContext(UseCasesContext);
  if (!v) throw new Error("useUseCases must be used inside <UseCasesProvider>");
  return v;
}
```
```tsx
// presentation/hooks/useAddExpense.ts — adapts a use case to TanStack Query (presentation concern)
export class FormValidationError extends Error { constructor(readonly issues: ValidationIssue[]) { super("validation failed"); } }

export function useAddExpense() {
  const { addExpense } = useUseCases();
  const qc = useQueryClient();
  return useMutation({
    mutationFn: async (draft: ExpenseDraft) => {
      const r = await addExpense(draft);
      if (!r.ok) throw new FormValidationError(r.error);
      return r.value;
    },
    onSuccess: () => qc.invalidateQueries({ queryKey: ["expenses"] }),
  });
}
```
```tsx
// presentation/components/AddExpenseForm.tsx — thin: collect input, show issues
export function AddExpenseForm() {
  const add = useAddExpense();
  const issues = add.error instanceof FormValidationError ? add.error.issues : [];
  const err = (f: string) => issues.find(i => i.field === f)?.message;
  return (
    <form onSubmit={(e) => { e.preventDefault(); const fd = new FormData(e.currentTarget);
      add.mutate({ amount: String(fd.get("amount")), category: String(fd.get("category")), spentOn: String(fd.get("spentOn")), description: String(fd.get("description") ?? "") }); }}>
      <label>Amount <input name="amount" inputMode="decimal" aria-invalid={!!err("amount")} /></label>
      {err("amount") && <span role="alert">{err("amount")}</span>}
      {/* category select from CATEGORIES, date input … */}
      <button disabled={add.isPending}>Save</button>
    </form>
  );
}
```
(TanStack Query stays in presentation: it is a UI-facing cache. The *rules* it calls live below.)

## Tests, per layer

```ts
// domain — pure, instant
it("rejects future dates", () => {
  const r = parseNewExpense({ amount: "5", category: "food", spentOn: "2026-10-07" }, "2026-10-06");
  expect(r).toEqual({ ok: false, error: [{ field: "spentOn", message: "Date can't be in the future" }] });
});
it("parses amounts without float math", () => expect(parseAmount("12.5")).toBe(1250));

// application — fakes, no DOM, no network
it("addExpense stores a valid expense", async () => {
  const gw = new FakeExpenseGateway();
  const add = makeAddExpense({ writer: gw, clock: { today: () => "2026-10-06" } });
  const r = await add({ amount: "12.50", category: "food", spentOn: "2026-10-05" });
  expect(r.ok && r.value.amount.cents).toBe(1250);
});

// infrastructure — contract + mapping (MSW)
expenseGatewayContract("fake", async () => new FakeExpenseGateway());
expenseGatewayContract("http", async () => new HttpExpenseGateway(new ApiClient(mswUrl, async () => "t")));

// presentation — component with FAKE use cases (no MSW needed)
it("shows a validation message", async () => {
  render(<UseCasesProvider useCases={testUseCases()}><AddExpenseForm /></UseCasesProvider>);
  await userEvent.type(screen.getByLabelText(/amount/i), "0");
  await userEvent.click(screen.getByRole("button", { name: /save/i }));
  expect(await screen.findByRole("alert")).toHaveTextContent(/greater than 0/i);
});
```
MSW remains valuable but moves to the infrastructure and full-integration tests; most tests need no network emulation at all.

## Enforce the layers

```js
// eslint.config.js (flat config) — pure core
export default [
  { files: ["src/domain/**"], rules: { "no-restricted-imports": ["error", { patterns: [
      { group: ["**/application/**", "**/infrastructure/**", "**/presentation/**", "**/app/**"], message: "domain is the innermost layer" },
      { group: ["react", "react-dom", "react-*", "zod", "aws-amplify", "@tanstack/*"], message: "domain must be framework-free" }] }] } },
  { files: ["src/application/**"], rules: { "no-restricted-imports": ["error", { patterns: [
      { group: ["**/infrastructure/**", "**/presentation/**", "**/app/**", "react", "react-dom", "@tanstack/*", "aws-amplify"], message: "application depends on domain only" }] }] } },
  { files: ["src/infrastructure/**"], rules: { "no-restricted-imports": ["error", { patterns: [
      { group: ["**/presentation/**", "**/app/**", "react", "react-dom"], message: "infrastructure must not know the UI" }] }] } },
];
```
Add `dependency-cruiser` for cycle detection if you like. Run both in CI ([../infra/07](../infra/07-cicd-github-actions.md)).

## Code that moves unchanged to Angular
`domain/`, `application/` and most of `infrastructure/` (Zod DTOs, gateway classes using an injected `fetch`/API client) can be **shared as a workspace package** (`packages/core`) consumed by both `web-react` and `web-angular`. Only `presentation/` and the composition file differ. That's a strong interview story: "I wrote the business logic once and bound it to two UI frameworks".

## SOLID check
| Principle | Evidence |
|---|---|
| SRP | components render; hooks adapt; use cases decide; gateways talk HTTP; domain validates |
| OCP | new data source (GraphQL, offline cache) = new gateway class + composition change |
| LSP | the gateway contract suite passes for `Fake` and `Http` |
| ISP | `ExpenseReader`/`ExpenseWriter` split; use cases take only what they need |
| DIP | use cases import `ports.ts`; `composition.ts` injects concrete gateways via `UseCasesProvider` |

## Pragmatism
For a 3-screen app this is more structure than strictly needed. State the trade-off: you'd adopt the full layering where logic is non-trivial, shared across frameworks, or changes with the API; small pure-presentation widgets can stay as components.

## Exercise
1. Implement the layers for "add expense + list + monthly totals" and move any `fetch`/validation out of existing components.
2. Write the 4 test types above; confirm domain and application tests run without jsdom (`// @vitest-environment node`).
3. Add the ESLint rules, then try importing `react` into `domain/` and see CI fail.
4. Extract `domain` + `application` + `infrastructure` into `packages/core` (npm workspaces) and consume it from a stub Angular app.

## Interview Q&A
- **Where does business logic live in your React apps?** In framework-free domain/application modules; components and hooks are thin adapters.
- **How do you test components without MSW?** Provide fake use cases through the context.
- **Why map DTOs to domain objects?** Decouple from API drift, enforce invariants at the boundary, use richer types (Money vs number).
- **Isn't this over-engineering for a UI?** Depends on complexity; here it enables reuse across React/Angular and fast tests, and I'd keep trivial presentational components simple.
- **Where does TanStack Query fit?** Presentation layer: server-state cache; it calls use cases.
