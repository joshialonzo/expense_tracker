# TypeScript 07 — SOLID in TypeScript (and a tiny DI container)

Architecture overview: [../fullstack/08](../fullstack/08-solid-and-clean-architecture.md). Applied to the UI: [../react/08](../react/08-clean-architecture-frontend.md), [../angular/06](../angular/06-clean-architecture-frontend.md); to Node/MCP: [../mcp/07](../mcp/07-mcp-and-assistant-as-adapters.md).

TypeScript is **structurally typed**, so interfaces are satisfied by shape: no `implements` needed, and the dependency direction stays inward for free. Functions are first-class, so a "port" can be an interface *or* a function type.

## S — Single Responsibility
Smell: one function that validates, calls the API, formats money, and shows a toast.

```ts
// ✘ four reasons to change
async function addExpense(form: FormData) {
  const amount = Number(form.get("amount"));
  if (!(amount > 0)) alert("bad amount");
  const res = await fetch("/api/expenses", { method: "POST", body: JSON.stringify({ amountCents: amount * 100 }) });
  document.querySelector("#total")!.textContent = `$${(await res.json()).amountCents / 100}`;
}
```
Split by reason to change:

```ts
// domain/money.ts      → rule
// application/addExpense.ts → intent (orchestration)
// infrastructure/HttpExpenseGateway.ts → transport
// presentation/AddExpenseForm.tsx → UI
```

## O — Open/Closed
Closed for modification, open for extension: add behavior by adding code, not editing a growing `switch`.

```ts
// ✘ every new provider edits this function
function insight(provider: string, data: Totals) {
  if (provider === "bedrock") { /* … */ } else if (provider === "azure") { /* … */ }
}

// ✔ strategy behind a port; add a provider = add a class + register it
interface InsightGenerator { generate(month: string, totals: Totals): Promise<Insight> }

const generators = {
  bedrock: (cfg: Cfg): InsightGenerator => new BedrockInsightGenerator(cfg),
  azure:   (cfg: Cfg): InsightGenerator => new AzureFoundryInsightGenerator(cfg),
} satisfies Record<string, (cfg: Cfg) => InsightGenerator>;

export const makeInsightGenerator = (name: keyof typeof generators, cfg: Cfg) => generators[name](cfg);
```
Discriminated unions give the *opposite* trade: adding a **variant** forces you to update every `switch` (compiler-enforced via `never`), which is right when the set is closed and you want exhaustiveness ([03](03-narrowing-and-discriminated-unions.md)). Choose polymorphism when you add *behaviors/implementations*, unions when you add *cases of data*.

## L — Liskov Substitution
Subtypes must honor the contract: same preconditions or weaker, same postconditions or stronger.

```ts
interface ExpenseGateway {
  /** Rejects with NotFoundError if absent or not owned. Never returns another user's data. */
  get(id: string): Promise<Expense>;
}
class FakeExpenseGateway implements ExpenseGateway { /* in-memory */ }
class HttpExpenseGateway implements ExpenseGateway { /* fetch */ }
```
Violations: a fake that returns `undefined` instead of throwing `NotFoundError`; an override that throws on input the base accepts; a `ReadonlyGateway extends ExpenseGateway` whose `add()` throws "not supported" (design smell; split the interface instead). Enforce with a **contract test** executed against every implementation:

```ts
export function expenseGatewayContract(name: string, make: () => Promise<ExpenseGateway>) {
  describe(`ExpenseGateway contract: ${name}`, () => {
    it("rejects with NotFoundError for an unknown id", async () => {
      await expect((await make()).get("nope")).rejects.toBeInstanceOf(NotFoundError);
    });
  });
}
expenseGatewayContract("fake", async () => new FakeExpenseGateway());
expenseGatewayContract("http (MSW)", async () => new HttpExpenseGateway(mswBaseUrl));
```

## I — Interface Segregation
Depend on the smallest capability you need.

```ts
// ✘ fat
interface ExpenseGateway { list(): …; get(): …; add(): …; remove(): …; export(): …; report(): … }

// ✔ role interfaces
interface ExpenseReader { list(q: ExpenseQuery): Promise<Page<Expense>>; get(id: string): Promise<Expense> }
interface ExpenseWriter { add(e: NewExpense): Promise<Expense>; remove(id: string): Promise<void> }
class HttpExpenseGateway implements ExpenseReader, ExpenseWriter { /* … */ }

// consumers ask for exactly what they use
const makeListExpenses = (deps: { reader: ExpenseReader }) => (q: ExpenseQuery) => deps.reader.list(q);

// or derive on the fly
type ReadOnly = Pick<ExpenseGateway, "list" | "get">;
```
Components should receive **narrow props**, not whole objects: `<ExpenseRow expense={e} onDelete={id => …} />` instead of passing the entire store.

## D — Dependency Inversion
High-level policy (use cases) depends on abstractions; details (fetch, Amplify, Bedrock SDK) implement them; a composition root connects them.

```ts
// application/addExpense.ts  — knows nothing about fetch/React/Angular
export interface AddExpenseDeps { writer: ExpenseWriter; clock: Clock }

export const makeAddExpense = ({ writer, clock }: AddExpenseDeps) =>
  async (input: AddExpenseInput): Promise<Result<Expense, ValidationIssue[]>> => {
    const parsed = parseNewExpense(input, clock.today());     // domain validation
    if (!parsed.ok) return parsed;
    return { ok: true, value: await writer.add(parsed.value) };
  };

// composition root (the ONLY place that names concrete classes)
const gateway = new HttpExpenseGateway(config.apiBase, tokenProvider);
export const useCases = { addExpense: makeAddExpense({ writer: gateway, clock: systemClock }) };
```
In tests, pass `{ writer: fake, clock: fixedClock }`. No mocking library, no module patching.

### Functions as ports (idiomatic and light)
```ts
type Clock = { today(): string };
type Fetcher = typeof fetch;                       // inject fetch → test without network
const makeGateway = (fetcher: Fetcher, base: string) => ({ /* uses fetcher */ });
```

## Constructor/factory injection vs a DI container
- **Default: manual injection** (factories or constructors + one composition file). It's explicit, type-checked, tree-shakeable.
- A container (InversifyJS, tsyringe) is warranted for big apps; Angular ships one (`inject`, providers: see [../angular/06](../angular/06-clean-architecture-frontend.md)). In React, **context** is the injection mechanism for the presentation layer ([../react/08](../react/08-clean-architecture-frontend.md)).
- Avoid decorators-and-reflection magic when plain functions suffice.

A 15-line typed registry if you need lazy singletons without a library:

```ts
type Factory<T> = (c: Container) => T;
export class Container {
  private cache = new Map<symbol, unknown>();
  private factories = new Map<symbol, Factory<unknown>>();
  register<T>(token: Token<T>, f: Factory<T>) { this.factories.set(token.id, f as Factory<unknown>); return this; }
  resolve<T>(token: Token<T>): T {
    if (!this.cache.has(token.id)) this.cache.set(token.id, (this.factories.get(token.id) ?? (() => { throw new Error(`No provider for ${token.name}`); }))(this));
    return this.cache.get(token.id) as T;
  }
}
export class Token<T> { readonly id = Symbol(this.name); declare readonly _t?: T; constructor(readonly name: string) {} }
export const EXPENSE_READER = new Token<ExpenseReader>("ExpenseReader");
```
(Note it is a **composition-root helper**; use cases must never receive the container itself, which would be the Service Locator anti-pattern.)

## Classes vs functions, OO vs FP
SOLID is phrased in OO terms but applies to modules and functions: SRP (small functions/modules), OCP (higher-order functions, strategy maps), ISP (narrow param types), DIP (inject dependencies as parameters). Prefer **immutable data + pure functions in the domain**, and small classes/closures at the edges. Be able to explain both.

## Anti-patterns to name
- **Service Locator** (`Container.resolve` inside business code) hides dependencies.
- **God service** (`ExpenseService` with 20 methods).
- **Leaky abstraction** (port returns `Response`, `AxiosError`, DynamoDB types).
- **Interface-per-class cargo cult** (an interface with one implementation and no reason to vary). Introduce ports at real seams: I/O, time, randomness, third-party SDKs, anything with a second implementation or a test need.
- **Anemic domain**: types with no behavior and all rules scattered in services/components.

## Exercise
1. Refactor the `addExpense` smell above into the four files, each with a test (domain: pure; application: fakes).
2. Write the gateway contract suite and run it against a fake and an MSW-backed HTTP gateway.
3. Add an `ESLint no-restricted-imports` rule making `src/domain` unable to import `react`, `axios`, `aws-amplify` or `@/infrastructure/*`.

## Interview Q&A
- **Which SOLID principle do you find most useful day to day?** DIP + ISP: they make testing cheap and swapping implementations safe; SRP follows.
- **How does structural typing change DI in TS?** No `implements` needed; fakes are just objects of the right shape, and inner layers needn't import outer ones.
- **Union types vs polymorphism for OCP?** Unions for closed sets of data cases with exhaustive checks; polymorphism/strategies for open sets of behaviors.
- **How do you avoid over-abstraction?** Introduce a port only at an I/O boundary or when there's a second implementation/test need.
- **How do you enforce architecture in TS?** ESLint boundaries/restricted imports, `dependency-cruiser`, Nx module boundaries, in CI.
