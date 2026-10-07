# React 05 — Routing, Protected Routes, Auth Flow and Project Structure

## Routing with React Router (v6/v7 data router)

```bash
npm i react-router-dom
```

```tsx
// router.tsx
export const router = createBrowserRouter([
  {
    path: "/",
    element: <Layout />,                       // renders <Outlet />
    errorElement: <RouteError />,
    children: [
      { index: true, element: <Navigate to="/expenses" replace /> },
      {
        element: <RequireAuth />,              // pathless layout route: guard
        children: [
          { path: "expenses", element: <ExpensesPage /> },
          { path: "expenses/:id", element: <ExpenseDetailPage /> },
          { path: "reports", lazy: () => import("./pages/ReportsPage") },   // code splitting
        ],
      },
      { path: "login", element: <LoginPage /> },
      { path: "*", element: <NotFound /> },
    ],
  },
]);

// main.tsx → <RouterProvider router={router} />
```

```tsx
function RequireAuth() {
  const { user, isLoading } = useAuth();
  const location = useLocation();
  if (isLoading) return <Spinner />;
  if (!user) return <Navigate to="/login" replace state={{ from: location }} />;
  return <Outlet />;
}
```

URL as state: keep filters in search params so links are shareable and the Back button works.

```tsx
const [params, setParams] = useSearchParams();
const month = params.get("month") ?? currentMonth();
const setMonth = (m: string) => setParams((p) => { p.set("month", m); return p; });
```

Route param: `const { id } = useParams<{ id: string }>()`; navigate imperatively with `useNavigate()`.

> **Security note**: client-side route guards are UX only. The API must enforce authorization on every request.

## Auth flow end-to-end (ties to AWS Cognito)

1. App loads → `AuthProvider` calls `fetchAuthSession()`; sets `isLoading` until resolved.
2. Unauthenticated → redirect to `/login` → `signInWithRedirect()` (Authorization Code + PKCE) ([aws/06](../aws/06-cognito-auth-jwt-oauth.md)).
3. Redirect back with `?code=` → SDK exchanges the code; `AuthProvider` sets `user`.
4. API client attaches `Authorization: Bearer <access token>`; refreshes automatically.
5. On `401` from API: try refresh once, else sign out → login.

Role-based UI: `const isAdmin = claims["cognito:groups"]?.includes("admin")`, hide buttons but rely on server checks.

## Environment config
Vite exposes only `VITE_*` variables to the browser bundle. **Never put secrets there**: everything in a SPA bundle is public. Values like API URL and Cognito client ID are fine. Use per-environment `.env.development`, `.env.production`, or inject at build in CI.

## Project structure (feature-based)

> **Preferred for this repo:** the layered structure in [08](08-clean-architecture-frontend.md) (`domain / application / infrastructure / presentation / app`), which keeps business rules framework-free and shareable with the Angular app. The feature-folder layout below is still useful *inside* `presentation/` (group components, hooks and pages by feature).


```
src/
  app/            router.tsx, providers.tsx, queryClient.ts
  api/            client.ts, schema.d.ts (generated from OpenAPI)
  auth/           AuthProvider.tsx, useAuth.ts, RequireAuth.tsx
  components/     ui/ (Button, Input, Modal): generic, no business logic
  features/
    expenses/
      components/ ExpenseList.tsx, ExpenseForm.tsx
      queries.ts  hooks
      types.ts
      index.ts    public API of the feature
    reports/
  pages/          route components that compose features
  lib/            money.ts, dates.ts
  test/           setup.ts, msw handlers
```
Principles: colocate by feature, one-way dependencies (pages → features → components/lib), avoid giant `utils.ts`, absolute imports with `@/` alias.

## Styling options (be able to defend one)
CSS Modules (zero-runtime, scoped), **Tailwind** (utility-first, fast), CSS-in-JS runtime (styled-components/emotion; has runtime cost, less favored now), component libs (MUI, Chakra, shadcn/ui built on Radix). For the JD ("HTML, CSS, JS knowledge", "responsive"): know Flexbox/Grid, media queries/container queries, `rem` units, mobile-first.

```css
.layout { display: grid; grid-template-columns: 1fr; gap: 1rem; }
@media (min-width: 768px) { .layout { grid-template-columns: 240px 1fr; } }
```

## Accessibility (a differentiator)
Semantic HTML (`button`, `nav`, `main`, `label`), `aria-*` only when semantics are insufficient, keyboard operability and visible focus, labels for every input, `role="alert"` for errors, color contrast. Test with axe (`jest-axe`/`@axe-core/playwright`) and Testing Library queries by role.

## Exercise
Add router, guard, and `/expenses?month=` URL-synced filters. Lazy-load `/reports`. Build the Cognito login flow and make the guard redirect back to the page the user wanted using `location.state.from`.

## Interview Q&A
- **How do you protect routes?** Guard layout route + server-side authorization; guards are UX only.
- **Where do you store tokens?** See [aws/06](../aws/06-cognito-auth-jwt-oauth.md): memory/HttpOnly cookie preferred; mitigate XSS.
- **How do you structure a growing React app?** By feature, with clear boundaries and public APIs; shared UI kit; no cross-feature deep imports.
- **How do you do code splitting?** `lazy()` + Suspense or router `lazy`, analyze the bundle (`vite-bundle-visualizer`).
- **SPA SEO / first paint?** SSR/SSG (Next.js/Remix) when needed; for an authenticated dashboard a SPA is fine.
