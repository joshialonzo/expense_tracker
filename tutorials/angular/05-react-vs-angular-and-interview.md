# Angular 05 — React vs Angular, and Interview Questions

## The big comparison (know this cold; it's a likely question)

| Aspect | React | Angular |
|---|---|---|
| Nature | **Library** for UI; compose your own stack | **Framework**, batteries included (router, forms, HTTP, DI, testing, CLI) |
| Language | JS/TS (JSX) | TypeScript-first (HTML templates + decorators) |
| Reactivity | Re-render components on state change; hooks; React Compiler optimizes | **Signals** (fine-grained) + RxJS; change detection (zone.js, moving to zoneless) |
| Templating | JSX: JS expressions in markup | HTML templates with directives/control flow (`@if`, `@for`) |
| State | `useState`, context, Zustand/Redux, TanStack Query | Signals, services (DI), NgRx, RxJS |
| DI | None built-in (context/props/hooks) | **First-class hierarchical DI** |
| Forms | RHF/Formik + Zod | Reactive Forms (built-in) |
| Routing | React Router/TanStack Router/Next | `@angular/router` (guards, resolvers, lazy loading) |
| HTTP | `fetch` + TanStack Query | `HttpClient` + interceptors |
| Learning curve | Small core, big ecosystem decisions | Larger up front, consistent conventions |
| Structure | Flexible (needs team conventions) | Opinionated (good for large teams) |
| Release cadence | Slower, gradual | Predictable semver, schematics `ng update` |
| Ecosystem | Largest; frameworks (Next.js, Remix) | Strong enterprise adoption (Google, banks, large orgs) |

Mapping of concepts:

| React | Angular |
|---|---|
| Component function + props | Component class + `input()` |
| Callback props | `output()` / EventEmitter |
| `useState` / `useMemo` / `useEffect` | `signal` / `computed` / `effect` |
| Context | DI (services/tokens) |
| Custom hook | Service or injectable function using `inject()` |
| `key` | `track` |
| Suspense/lazy | `@defer` / `loadComponent` |
| Error boundary | `ErrorHandler` class / `@defer` error blocks |
| React Router loaders | Resolvers / guards |
| Axios interceptor / fetch wrapper | `HttpInterceptorFn` |
| MSW + Testing Library | `HttpTestingController` + Angular Testing Library |

**How to choose (the senior answer)**: "Both solve the same problems. I'd pick Angular for large teams that benefit from enforced structure, built-in DI/forms and long-term maintainability; React when flexibility, a huge ecosystem, or SSR frameworks matter (Next.js). The architecture (typed API contract, server-state caching, auth, tests) matters more than the choice, and my skills transfer: I've built the same expense tracker in both against one OpenAPI backend."

## Questions to expect (Angular, nice-to-have)
1. Standalone components vs NgModules; why the shift?
2. Signals vs RxJS; when to use which; interop (`toSignal`, `toObservable`).
3. Change detection: Default vs OnPush; what triggers a check; zone.js role.
4. DI: providers, `providedIn`, hierarchical injectors, injection tokens, `inject()`.
5. Component communication: inputs/outputs, services, `model()`, content projection (`<ng-content>`), `ViewChild`.
6. Lifecycle hooks and their use.
7. Pipes: pure vs impure; `async` pipe benefits.
8. Directives: structural vs attribute; building a custom attribute directive.
9. Routing: guards, resolvers, lazy loading, params, `withComponentInputBinding`.
10. Reactive vs template-driven forms; custom validators.
11. HttpClient interceptors; error handling/retry.
12. RxJS flattening operators and memory-leak avoidance.
13. Testing: TestBed, `fakeAsync`, mocking services, `HttpTestingController`.
14. Performance: OnPush, `track`, `@defer`, lazy routes, bundle analysis, SSR hydration.
15. Security: sanitization, XSRF, CSP.
16. Upgrading Angular: `ng update`, migrations (to standalone, to control flow, to signals).

### Mini drills
**Attribute directive** — highlight rows over budget:
```ts
@Directive({ selector: "[appOverBudget]", standalone: true, host: { "[class.over]": "over()" } })
export class OverBudgetDirective {
  spent = input.required<number>();
  limit = input.required<number>();
  over = computed(() => this.spent() > this.limit());
}
```
**Pure pipe** — money formatting:
```ts
@Pipe({ name: "cents", standalone: true })
export class CentsPipe implements PipeTransform {
  transform(cents: number, currency = "USD") {
    return new Intl.NumberFormat("en-US", { style: "currency", currency }).format(cents / 100);
  }
}
```
**Content projection**:
```html
<!-- card.component.html -->
<div class="card"><h3><ng-content select="[title]" /></h3><ng-content /></div>
<!-- usage -->
<app-card><span title>Totals</span><app-expense-totals /></app-card>
```
**Spot the bug**:
```ts
ngOnInit() { this.form.valueChanges.subscribe(v => this.search(v)); }   // never unsubscribed → leak + duplicates
```
Fix with `takeUntilDestroyed()` or `toSignal`.

## Practice plan
1. Build the Angular app ([01](01-fundamentals-components-and-signals.md)–[04](04-auth-state-and-testing.md)) against the existing backend.
2. Write the concept mapping above in your own words; do a React↔Angular translation of one feature (search + list + create) and note differences.
3. Deploy both SPAs to different CloudFront paths/buckets from the same CI/CD workflow matrix ([../aws/08](../aws/08-cicd-observability-and-cost.md)).

## Honest positioning for the interview
If your Angular is lighter than your React: lead with React depth, show you can map concepts across (DI, signals/RxJS, forms), and mention the working Angular version of this project. Interviewers value transferable understanding and honesty over bluffing.
