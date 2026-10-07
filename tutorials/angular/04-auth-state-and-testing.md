# Angular 04 — Auth with Cognito, State Management, Testing and Performance

## Auth service (Cognito via Amplify, signals-based)

```ts
// core/auth.service.ts
import { Injectable, computed, signal } from "@angular/core";
import { fetchAuthSession, getCurrentUser, signInWithRedirect, signOut as amplifySignOut } from "aws-amplify/auth";

@Injectable({ providedIn: "root" })
export class AuthService {
  private _user = signal<{ sub: string; email?: string } | null>(null);
  private _ready = signal(false);

  user = this._user.asReadonly();
  isReady = this._ready.asReadonly();
  isAuthenticated = computed(() => this._user() !== null);

  async init() {                                     // call via provideAppInitializer before routing
    try {
      const u = await getCurrentUser();
      this._user.set({ sub: u.userId });
    } catch { this._user.set(null); }
    this._ready.set(true);
  }

  signIn()  { return signInWithRedirect(); }         // Authorization Code + PKCE via Hosted UI
  async signOut() { await amplifySignOut(); this._user.set(null); }

  async accessToken(): Promise<string | undefined> {
    const { tokens } = await fetchAuthSession();     // refreshes when needed
    return tokens?.accessToken?.toString();
  }
}
```
Amplify configuration is the same as in [../aws/06](../aws/06-cognito-auth-jwt-oauth.md) (`Amplify.configure(...)` in `main.ts`). Alternatives: `angular-oauth2-oidc`, `angular-auth-oidc-client`. The interceptor and guard from [02](02-services-di-http-and-routing.md) consume this service. Make the guard wait for `isReady` (or use an app initializer) to avoid a login flash on refresh.

Role-based UI with a structural check: `@if (auth.hasGroup('admin')) { … }`, but authorization is enforced by the API.

> **Clean-architecture note:** the signal store should call **use cases**, not `HttpClient`-based services directly ([06](06-clean-architecture-frontend.md)).

## State management options

| Need | Tool |
|---|---|
| Local component state | `signal` |
| Shared state in a feature | **Signal-based service** (below) |
| Server state caching | TanStack Query (Angular adapter) or hand-rolled with `resource`/RxJS `shareReplay` |
| Large apps, strict patterns, devtools | **NgRx** (Redux-style: actions, reducers, effects, selectors) or **NgRx SignalStore** |

Signal store service (lightweight, often enough):

```ts
@Injectable({ providedIn: "root" })
export class ExpenseStore {
  private api = inject(ExpenseApi);
  private _items = signal<Expense[]>([]);
  private _loading = signal(false);

  items = this._items.asReadonly();
  loading = this._loading.asReadonly();
  total = computed(() => this._items().reduce((s, e) => s + e.amountCents, 0));
  byCategory = computed(() => Object.entries(
    this._items().reduce<Record<string, number>>((m, e) => ({ ...m, [e.category]: (m[e.category] ?? 0) + e.amountCents }), {})));

  load(month: string) {
    this._loading.set(true);
    this.api.list({ from: `${month}-01`, to: `${month}-31` })
      .pipe(finalize(() => this._loading.set(false)))
      .subscribe(p => this._items.set(p.items));
  }
  add(e: Expense) { this._items.update(list => [e, ...list]); }
  remove(id: string) {
    const prev = this._items();
    this._items.update(list => list.filter(e => e.id !== id));         // optimistic
    this.api.remove(id).subscribe({ error: () => this._items.set(prev) });   // rollback
  }
}
```

## Testing

### Component test with TestBed

```ts
describe("ExpenseListComponent", () => {
  let fixture: ComponentFixture<ExpenseListComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({ imports: [ExpenseListComponent] }).compileComponents();
    fixture = TestBed.createComponent(ExpenseListComponent);
    fixture.componentRef.setInput("expenses", [
      { id: "1", amountCents: 1250, currency: "USD", category: "food", description: "Lunch", date: "2026-10-01" },
    ]);
    fixture.detectChanges();
  });

  it("renders expenses and total", () => {
    const el: HTMLElement = fixture.nativeElement;
    expect(el.textContent).toContain("Lunch");
    expect(el.querySelector("strong")?.textContent).toContain("$12.50");
  });

  it("emits deleted when delete clicked", () => {
    const spy = jasmine.createSpy();            // or vi.fn() / jest.fn() depending on the runner
    fixture.componentInstance.deleted.subscribe(spy);
    fixture.nativeElement.querySelector("button").click();
    expect(spy).toHaveBeenCalledWith("1");
  });
});
```

### Service/HTTP test

```ts
beforeEach(() => TestBed.configureTestingModule({ providers: [provideHttpClient(), provideHttpClientTesting()] }));

it("lists expenses", () => {
  const api = TestBed.inject(ExpenseApi);
  const http = TestBed.inject(HttpTestingController);
  api.list({ category: "food" }).subscribe(p => expect(p.items.length).toBe(1));
  const req = http.expectOne(r => r.url.endsWith("/expenses") && r.params.get("category") === "food");
  expect(req.request.method).toBe("GET");
  req.flush({ items: [{ id: "1" }], nextCursor: null });
  http.verify();
});
```
Async time control with `fakeAsync`/`tick()`/`flush()`; DOM-centric testing with **Angular Testing Library** (`@testing-library/angular`) mirrors React Testing Library's philosophy. E2E: **Playwright** (same tests as React app: the UI contract is the same).

## Performance
- `OnPush` + signals; `track` in `@for`; `@defer` for heavy widgets; lazy routes.
- Avoid heavy function calls in templates (use `computed`/pure pipes).
- Build optimizations are on by default for `ng build` (AOT, tree-shaking, minification); inspect with `source-map-explorer`.
- `NgOptimizedImage` for images; SSR/prerender (Angular SSR + hydration) if SEO/first paint matters.
- Zoneless change detection is emerging with signals (check your Angular version).

## Security (Angular specifics)
Templates auto-sanitize bound HTML/URLs (`DomSanitizer` bypass methods are dangerous); built-in XSRF protection for cookie-based auth (`withXsrfConfiguration`); CSP; don't store secrets in `environment.ts` (it ships to the browser).

## Exercise
Implement `AuthService` + `ExpenseStore`, wire the list page to the store, and write the component and HTTP tests above. Add a Playwright test shared with the React app.

## Interview Q&A
- **How do you manage state in Angular?** Signals for local/feature state, services as stores, NgRx for large/complex state; server cache with a query library.
- **How do you test an HTTP service?** `HttpTestingController`.
- **How do you protect routes?** Functional guards + server-side authorization.
- **Biggest performance wins?** OnPush/signals, lazy loading, `@defer`, avoiding template function calls.
