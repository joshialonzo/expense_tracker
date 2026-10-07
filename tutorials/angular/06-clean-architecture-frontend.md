# Angular 06 — Clean Architecture and SOLID in Angular

Principles: [../fullstack/08](../fullstack/08-solid-and-clean-architecture.md). The React twin is [../react/08](../react/08-clean-architecture-frontend.md). **`domain/` and `application/` are the same TypeScript** (ideally the shared `packages/core` workspace package); this tutorial shows the Angular-specific parts: DI-based ports, infrastructure services, and signal-based presentation.

> **As implemented in [`web-angular/`](../../web-angular)** (38 tests): `domain/` and `application/` are **byte-for-byte the same TypeScript as `web-react`** (interfaces + `makeXxx` factory use cases, so the application layer has zero Angular imports). Angular's DI is used only at the edges: `di.ts` declares `InjectionToken`s (`USE_CASES`, `AUTH_SESSION`, `RUNTIME_CONFIG`, `CLOCK`), and `app.config.ts` is the composition root that builds the use cases from `HttpClient`-based gateways. That is the "Option A" below taken to its conclusion. The abstract-class ports shown next are the more Angular-idiomatic alternative; the trade-off is that the application layer then imports `@angular/core`. A script (`npm run check:arch`) enforces the dependency rule, and it already caught the store importing the infrastructure clock directly (fixed with a `CLOCK` token).

## Angular already gives you the machinery
Angular's **hierarchical DI** is a built-in dependency-inversion container, and **abstract classes work as injection tokens**. So:

- **Ports** = abstract classes (or `InjectionToken`s) in `application/`.
- **Adapters** = `@Injectable()` classes in `infrastructure/`, bound in `app.config.ts` (the composition root).
- **Use cases** = small `@Injectable()` classes (or plain factories wrapped in providers).
- **Presentation** = components + signal-based facades/stores that call use cases.

## Layout

```
web-angular/src/app/
  domain/            money.ts expense.ts category.ts summary.ts errors.ts result.ts     (pure TS, same as React)
  application/
    ports/expense-gateway.ts            # abstract classes = DI tokens
    ports/clock.ts
    use-cases/add-expense.ts  list-expenses.ts  monthly-totals.ts  remove-expense.ts
  infrastructure/
    http/http-expense-gateway.ts  http/dto.ts  http/runtime-config.ts
    auth/auth.interceptor.ts  auth/auth.service.ts
    system/system-clock.ts
    fake/fake-expense-gateway.ts
  presentation/
    expenses/expenses.page.ts  expense-form.component.ts  expenses.store.ts
    shell/…  routes.ts
  app.config.ts      # composition root (providers)
```

## Ports as abstract classes (DI tokens)

```ts
// application/ports/expense-gateway.ts
import type { Expense, NewExpense } from "../../domain/expense";

export type Page<T> = { items: T[]; nextCursor: string | null };
export type ExpenseQuery = { from: string; to: string; category?: string; cursor?: string };

/** ISP: two roles. A concrete class may implement both. */
export abstract class ExpenseReader {
  abstract list(q: ExpenseQuery): Promise<Page<Expense>>;
}
export abstract class ExpenseWriter {
  abstract add(e: NewExpense): Promise<Expense>;
  abstract remove(id: string): Promise<void>;
}
```
```ts
// application/ports/clock.ts
export abstract class Clock { abstract today(): string }
```
Because they are real classes at runtime, `inject(ExpenseWriter)` works with no `InjectionToken` boilerplate, while the application layer still has **no import of Angular's HttpClient or any adapter**. (`@Injectable` isn't needed on the abstract class; the *application layer may import `inject` from `@angular/core`*, a deliberate, tiny framework dependency. If you want it fully framework-free, use plain constructor parameters and wire with `useFactory` providers, shown below.)

## Use cases

Option A: framework-free class + factory provider (purest):
```ts
// application/use-cases/add-expense.ts
export class AddExpense {
  constructor(private readonly writer: ExpenseWriter, private readonly clock: Clock) {}

  async execute(draft: ExpenseDraft): Promise<Result<Expense, ValidationIssue[]>> {
    const parsed = parseNewExpense(draft, this.clock.today());
    if (!parsed.ok) return parsed;
    return { ok: true, value: await this.writer.add(parsed.value) };
  }
}
```
Option B: Angular-idiomatic (`providedIn: "root"` + `inject`):
```ts
@Injectable({ providedIn: "root" })
export class MonthlyTotals {
  private reader = inject(ExpenseReader);
  async execute(month: string) { /* same body as the React use case */ }
}
```
Pick one style and be consistent; A keeps `application/` free of `@angular/*`, B is shorter. Both are DIP-compliant: the class asks for the **abstraction**.

## Infrastructure adapters

```ts
// infrastructure/http/http-expense-gateway.ts
@Injectable()
export class HttpExpenseGateway implements ExpenseReader, ExpenseWriter {
  private http = inject(HttpClient);
  private cfg = inject(RUNTIME_CONFIG);

  async list(q: ExpenseQuery): Promise<Page<Expense>> {
    let params = new HttpParams().set("from", q.from).set("to", q.to);
    if (q.category) params = params.set("category", q.category);
    if (q.cursor) params = params.set("cursor", q.cursor);
    const raw = await firstValueFrom(this.http.get<unknown>(`${this.cfg.apiBase}/expenses`, { params }));
    const dto = PageDto.parse(raw);                       // same Zod DTOs/mappers as React (anti-corruption layer)
    return { items: dto.items.map(toDomain), nextCursor: dto.nextCursor };
  }
  async add(e: NewExpense) {
    return toDomain(ExpenseDto.parse(await firstValueFrom(this.http.post(`${this.cfg.apiBase}/expenses`, toCreateBody(e)))));
  }
  async remove(id: string) { await firstValueFrom(this.http.delete(`${this.cfg.apiBase}/expenses/${id}`)); }
}
```
Where you want RxJS semantics (cancel stale requests, retry, debounce) keep `Observable` in the port instead (`list(q): Observable<Page<Expense>>`); the dependency rule is unaffected. Promise-based ports make the use cases identical to React's, which is the reuse goal; choose consciously and document it.

HTTP concerns that are *not* business logic stay in infrastructure: the auth **interceptor** ([02](02-services-di-http-and-routing.md)), error mapping (401 → `UnauthorizedError`), retries.

## Composition root: `app.config.ts`

```ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding()),
    provideHttpClient(withInterceptors([authInterceptor])),

    // ports → adapters (swap any line to change an implementation)
    HttpExpenseGateway,
    { provide: ExpenseReader, useExisting: HttpExpenseGateway },
    { provide: ExpenseWriter, useExisting: HttpExpenseGateway },
    { provide: Clock, useClass: SystemClock },

    // use cases (Option A style)
    { provide: AddExpense, useFactory: () => new AddExpense(inject(ExpenseWriter), inject(Clock)) },
  ],
};
```
Tests and Storybook provide `{ provide: ExpenseWriter, useClass: FakeExpenseGateway }` instead. No other code changes.

## Presentation: signal store/facade + thin components

```ts
// presentation/expenses/expenses.store.ts
@Injectable({ providedIn: "root" })
export class ExpensesStore {
  private listUC = inject(ListExpenses);
  private addUC = inject(AddExpense);
  private removeUC = inject(RemoveExpense);

  readonly items = signal<Expense[]>([]);
  readonly loading = signal(false);
  readonly issues = signal<ValidationIssue[]>([]);
  readonly totals = computed(() => totalsByCategory(this.items()));     // domain function

  async load(month: string) {
    this.loading.set(true);
    try { this.items.set((await this.listUC.execute(month)).items); }
    finally { this.loading.set(false); }
  }
  async add(draft: ExpenseDraft) {
    const r = await this.addUC.execute(draft);
    if (r.ok) { this.items.update(l => [r.value, ...l]); this.issues.set([]); } else this.issues.set(r.error);
  }
  async remove(id: string) {
    const prev = this.items();
    this.items.update(l => l.filter(e => e.id !== id));                // optimistic
    try { await this.removeUC.execute(id); } catch { this.items.set(prev); }
  }
}
```
```ts
// presentation/expenses/expense-form.component.ts
@Component({
  selector: "app-expense-form", standalone: true, changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [FormsModule],
  template: `
    <form (ngSubmit)="submit()">
      <label>Amount <input name="amount" [(ngModel)]="amount" [attr.aria-invalid]="!!err('amount')" /></label>
      @if (err('amount'); as m) { <span role="alert">{{ m }}</span> }
      <!-- category select over CATEGORIES, date input -->
      <button type="submit">Save</button>
    </form>`,
})
export class ExpenseFormComponent {
  private store = inject(ExpensesStore);
  amount = ""; category = "food"; spentOn = new Date().toISOString().slice(0, 10);
  err = (f: string) => this.store.issues().find(i => i.field === f)?.message;
  submit() { void this.store.add({ amount: this.amount, category: this.category, spentOn: this.spentOn }); }
}
```
Domain validation (`parseNewExpense`) is reused verbatim. For heavily dynamic forms, Reactive Forms ([03](03-rxjs-and-forms.md)) can call the same domain parser inside a custom validator (`(c) => parseAmount(c.value) ? null : { amount: true }`) so rules aren't re-implemented in the form layer.

## Tests

```ts
// application: no TestBed needed
it("rejects future dates", async () => {
  const uc = new AddExpense(new FakeExpenseGateway(), { today: () => "2026-10-06" });
  const r = await uc.execute({ amount: "5", category: "food", spentOn: "2026-10-07" });
  expect(r.ok).toBe(false);
});

// presentation: swap the adapter with a provider, no HTTP at all
beforeEach(() => TestBed.configureTestingModule({
  providers: [
    FakeExpenseGateway,
    { provide: ExpenseReader, useExisting: FakeExpenseGateway },
    { provide: ExpenseWriter, useExisting: FakeExpenseGateway },
    { provide: Clock, useValue: { today: () => "2026-10-06" } },
    { provide: AddExpense, useFactory: () => new AddExpense(inject(ExpenseWriter), inject(Clock)) },
  ],
}));

// infrastructure: HttpTestingController + shared gateway contract suite (same one as React)
expenseGatewayContract("fake", async () => new FakeExpenseGateway());
```

## Enforce boundaries
Same idea as React: ESLint `no-restricted-imports` per folder (domain/application must not import `@angular/*` (Option A), `rxjs` (if promise ports), `infrastructure`, `presentation`), or **Nx** `@nx/enforce-module-boundaries` with tags (`layer:domain`, `layer:application`, …) if you adopt Nx. Run in CI ([../infra/07](../infra/07-cicd-github-actions.md)).

## React ↔ Angular mapping (clean-architecture view)
| Concept | React | Angular |
|---|---|---|
| Port | interface in `ports.ts` | abstract class (DI token) |
| Adapter binding | `buildUseCases()` + context | `providers` in `app.config.ts` |
| Use case | factory function | class (factory/`inject`) |
| UI state facade | hooks + TanStack Query | signal store service |
| Fake for tests | pass fake use cases to provider | override providers in TestBed |
| Shared code | `domain`, `application`, DTO mappers | **identical** (workspace package) |

## SOLID check
SRP (component/store/use case/gateway), OCP (new gateway = new provider), LSP (contract suite), ISP (`ExpenseReader` vs `ExpenseWriter`), DIP (abstract-class ports resolved by the injector).

## Exercise
1. Move `domain` and `application` into `packages/core` shared by `web-react` and `web-angular`; make both apps pass the same use-case tests.
2. Implement `ExpensesStore` with a fake gateway and show the SPA running with no backend.
3. Replace `HttpExpenseGateway` with an offline-first gateway (IndexedDB + sync) by changing only providers.

## Interview Q&A
- **How does Angular DI relate to dependency inversion?** The injector is the composition mechanism: classes request abstractions, providers bind implementations; swapping is configuration.
- **Why abstract classes as tokens?** They exist at runtime (unlike interfaces) so they can be injection keys while still defining the contract.
- **Where does business logic live?** Domain/application, not components or the store; the store only adapts use cases to signals.
- **How do you avoid Angular leaking into the core?** Option A: plain classes + factory providers; lint rules forbidding `@angular/*` imports in `domain/` and `application/`.
