# Angular 02 — Services, Dependency Injection, HttpClient and Routing

> **Clean-architecture note:** `ExpenseApi` below is an **infrastructure adapter**. In the layered design ([06](06-clean-architecture-frontend.md)) components/stores depend on an abstract `ExpenseReader`/`ExpenseWriter` port (an abstract class used as a DI token), and `app.config.ts` binds the HTTP implementation.

## Dependency injection (a big Angular differentiator)
Angular has a **built-in hierarchical DI container**. Services are classes registered with an injector; components ask for them.

```ts
// core/expense-api.service.ts
import { Injectable, inject } from "@angular/core";
import { HttpClient, HttpParams } from "@angular/common/http";
import { Observable } from "rxjs";
import { environment } from "../../environments/environment";

@Injectable({ providedIn: "root" })          // app-wide singleton, tree-shakable
export class ExpenseApi {
  private http = inject(HttpClient);          // modern: inject() function (constructor injection still valid)
  private base = environment.apiUrl;

  list(filters: { from?: string; to?: string; category?: string; cursor?: string }): Observable<Page<Expense>> {
    let params = new HttpParams();
    for (const [k, v] of Object.entries(filters)) if (v) params = params.set(k, v);
    return this.http.get<Page<Expense>>(`${this.base}/expenses`, { params });
  }
  create(body: CreateExpense) { return this.http.post<Expense>(`${this.base}/expenses`, body); }
  remove(id: string)          { return this.http.delete<void>(`${this.base}/expenses/${id}`); }
}
```
Scopes: `providedIn: 'root'` (singleton), component-level `providers: [...]` (one per component instance), route-level providers. **Injection tokens** (`InjectionToken<T>`) inject non-class values (config, interfaces). Swap implementations in tests with `TestBed.configureTestingModule({ providers: [{ provide: ExpenseApi, useValue: fake }] })`.

React comparison: DI replaces much of what you do with context/props/custom hooks, and gives constructor-level testability.

## App bootstrap and providers

```ts
// app.config.ts
import { ApplicationConfig, provideZoneChangeDetection } from "@angular/core";
import { provideRouter, withComponentInputBinding } from "@angular/router";
import { provideHttpClient, withInterceptors } from "@angular/common/http";
import { routes } from "./app.routes";
import { authInterceptor } from "./core/auth.interceptor";

export const appConfig: ApplicationConfig = {
  providers: [
    provideZoneChangeDetection({ eventCoalescing: true }),
    provideRouter(routes, withComponentInputBinding()),   // route params → component inputs
    provideHttpClient(withInterceptors([authInterceptor])),
  ],
};
// main.ts: bootstrapApplication(AppComponent, appConfig)
```

## HTTP interceptors (functional)

```ts
// core/auth.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from "@angular/common/http";
import { inject } from "@angular/core";
import { catchError, from, switchMap, throwError } from "rxjs";
import { AuthService } from "./auth.service";

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthService);
  if (!req.url.startsWith(environment.apiUrl)) return next(req);       // don't leak tokens to other origins

  return from(auth.accessToken()).pipe(
    switchMap(token => next(token ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }) : req)),
    catchError((err: HttpErrorResponse) => {
      if (err.status === 401) auth.signOut();
      return throwError(() => err);
    }),
  );
};
```
Interceptors are the Angular analogue of Axios interceptors / FastAPI middleware: auth headers, error handling, logging, retry, loading indicators.

## Using the service in a component (signals + RxJS interop)

```ts
@Component({ /* … */ })
export class ExpensesPage {
  private api = inject(ExpenseApi);

  month = signal(currentMonth());
  // reactive: when month changes, re-run the query
  expenses = toSignal(
    toObservable(this.month).pipe(
      switchMap(m => this.api.list({ from: `${m}-01`, to: `${m}-31` })),   // switchMap cancels the stale request
      map(p => p.items),
    ),
    { initialValue: [] as Expense[] },
  );
}
```
Newer Angular versions add `resource()`/`httpResource()` for signal-based async data (still evolving: check your version). Third-party option: **TanStack Query for Angular** gives the same caching model as in [../react/04](../react/04-data-fetching-tanstack-query.md).

## Routing

```ts
// app.routes.ts
import { Routes } from "@angular/router";
import { authGuard } from "./core/auth.guard";

export const routes: Routes = [
  { path: "", pathMatch: "full", redirectTo: "expenses" },
  { path: "login", loadComponent: () => import("./pages/login.page").then(m => m.LoginPage) },
  {
    path: "",
    canActivate: [authGuard],
    children: [
      { path: "expenses", loadComponent: () => import("./pages/expenses.page").then(m => m.ExpensesPage) },
      { path: "expenses/:id", loadComponent: () => import("./pages/expense-detail.page").then(m => m.ExpenseDetailPage),
        resolve: { expense: expenseResolver } },
      { path: "reports", loadChildren: () => import("./reports/reports.routes").then(m => m.REPORT_ROUTES) },
    ],
  },
  { path: "**", loadComponent: () => import("./pages/not-found.page").then(m => m.NotFoundPage) },
];
```

```ts
// core/auth.guard.ts
import { CanActivateFn, Router } from "@angular/router";
export const authGuard: CanActivateFn = (route, state) => {
  const auth = inject(AuthService);
  const router = inject(Router);
  return auth.isAuthenticated() ? true : router.createUrlTree(["/login"], { queryParams: { returnUrl: state.url } });
};
```
`loadComponent`/`loadChildren` = **lazy loading** (code splitting). Route params with `withComponentInputBinding()`:

```ts
id = input.required<string>();     // from /expenses/:id
```
Template: `<a routerLink="/expenses" routerLinkActive="active">Expenses</a>` plus `<router-outlet />`.

## Environment config
`src/environments/environment.ts` (+ `.prod.ts` with file replacement) or runtime config loaded via `APP_INITIALIZER`/`provideAppInitializer` from `/config.json` (so the same build deploys to many environments, a good AWS S3/CloudFront practice).

## Deploying to AWS
`ng build` → `dist/web-angular/browser` → same S3 + CloudFront flow as the React app ([../aws/02](../aws/02-s3-and-cloudfront.md)), including the 403/404 → `index.html` rewrite and cache headers.

## Exercise
Build `ExpenseApi`, the `authInterceptor`, the guard and lazy routes. Run it against the same FastAPI/Gin backend as the React app to prove the API contract is frontend-agnostic.

## Interview Q&A
- **Explain Angular DI and `providedIn: 'root'`.** Hierarchical injectors; root-provided tree-shakable singleton.
- **Interceptor use cases?** Auth header, global errors, retry, caching, loading state.
- **How do you lazy load?** `loadComponent` / `loadChildren` with dynamic import.
- **Guard types?** `canActivate`, `canMatch`, `canDeactivate`, resolvers for pre-fetching.
- **Why `switchMap` for search/route changes?** Cancels the previous inner request: no race conditions.
