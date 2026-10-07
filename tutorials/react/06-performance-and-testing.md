# React 06 — Performance and Testing

## Performance: a method, not a bag of tricks

1. **Measure**: React DevTools Profiler (why did this render?), Chrome Performance panel, Lighthouse / Web Vitals (LCP, INP, CLS).
2. **Find the category**: too many renders? expensive renders? too much JS? slow network? layout thrash?
3. **Fix the biggest one**, re-measure.

### Rendering costs
- State placed too high re-renders huge subtrees → **move state down** or split components.
- Pass `children` through a stateful wrapper: children elements keep identity and don't re-render when the wrapper's state changes.
- `React.memo` + stable props (`useCallback`/`useMemo`) for expensive pure children ([02](02-hooks-effects-and-refs.md)).
- Context value churn → split/memoize contexts, or use selectors (Zustand).
- **Long lists**: virtualize (`@tanstack/react-virtual`, `react-window`); paginate server-side.
- **Heavy computations**: `useMemo`, move to a Web Worker, or `useDeferredValue`/`useTransition` for responsiveness.
- Avoid creating components inside components (remounts every render).
- Don't use array index keys for dynamic lists.

### Bundle and network
- Route-level `lazy()` splitting; dynamic `import()` for heavy libs (charts, PDF).
- Analyze with a bundle visualizer; replace heavy deps (`moment` → `date-fns`/`Intl`), tree-shakable imports.
- Compress (Brotli at CloudFront), cache hashed assets for a year ([aws/02](../aws/02-s3-and-cloudfront.md)), `preconnect` to the API origin, image optimization (`srcset`, `loading="lazy"`, WebP/AVIF).
- Server state: `staleTime` to avoid refetch storms; prefetch on hover (`queryClient.prefetchQuery`).

### Web Vitals quick glossary
- **LCP** (Largest Contentful Paint) < 2.5 s: loading speed.
- **INP** (Interaction to Next Paint) < 200 ms: responsiveness (replaced FID).
- **CLS** (Cumulative Layout Shift) < 0.1: visual stability (set image dimensions, reserve space).

## Testing pyramid for the frontend

| Layer | Tool | What |
|---|---|---|
| Unit | Vitest | pure functions, reducers, hooks |
| Component/integration | **React Testing Library** + MSW | behavior through the DOM like a user |
| E2E | Playwright (or Cypress) | critical flows against a real/staged backend |
| Types/lint | `tsc`, ESLint | static |

Philosophy of Testing Library: **test behavior, not implementation**. Query by role/label/text (what users perceive), not by class names or component internals.

### Setup (Vitest)

```ts
// vite.config.ts
export default defineConfig({
  plugins: [react()],
  test: { environment: "jsdom", setupFiles: "./src/test/setup.ts", globals: true, coverage: { reporter: ["text", "lcov"] } },
});
// src/test/setup.ts
import "@testing-library/jest-dom/vitest";
import { afterAll, afterEach, beforeAll } from "vitest";
import { server } from "./server";
beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

### Component test with MSW

```tsx
// src/test/handlers.ts
import { http, HttpResponse } from "msw";
export const handlers = [
  http.get("*/expenses", () => HttpResponse.json({ items: [
    { id: "1", amountCents: 1250, currency: "USD", category: "food", description: "Lunch", createdAt: "2026-10-01T00:00:00Z" },
  ], nextCursor: null })),
  http.post("*/expenses", async ({ request }) => HttpResponse.json({ id: "2", ...(await request.json()) }, { status: 201 })),
];
// src/test/server.ts → export const server = setupServer(...handlers)
```

```tsx
// ExpensesPage.test.tsx
import { render, screen, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";

function renderWithProviders(ui: React.ReactElement) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  return render(<QueryClientProvider client={client}><MemoryRouter>{ui}</MemoryRouter></QueryClientProvider>);
}

it("lists expenses and adds a new one", async () => {
  const user = userEvent.setup();
  renderWithProviders(<ExpensesPage />);

  expect(await screen.findByText(/lunch/i)).toBeInTheDocument();      // findBy* = async wait

  await user.type(screen.getByLabelText(/amount/i), "9.99");
  await user.selectOptions(screen.getByLabelText(/category/i), "food");
  await user.click(screen.getByRole("button", { name: /save/i }));

  expect(await screen.findByRole("status")).toHaveTextContent(/saved/i);
});

it("shows an error when the API fails", async () => {
  server.use(http.get("*/expenses", () => new HttpResponse(null, { status: 500 })));
  renderWithProviders(<ExpensesPage />);
  expect(await screen.findByRole("alert")).toBeInTheDocument();
});
```

Query priority: `getByRole` > `getByLabelText` > `getByText` > `getByTestId` (last resort). `getBy` throws if missing, `queryBy` returns null (assert absence), `findBy` awaits.

### Testing hooks and reducers

```ts
import { renderHook, act } from "@testing-library/react";
it("toggles", () => {
  const { result } = renderHook(() => useToggle());
  act(() => result.current.toggle());
  expect(result.current.on).toBe(true);
});
```

### E2E with Playwright

```ts
import { test, expect } from "@playwright/test";
test("user can add an expense", async ({ page }) => {
  await page.goto("/");
  await page.getByLabel("Amount").fill("12.50");
  await page.getByRole("button", { name: "Save" }).click();
  await expect(page.getByText("12.50")).toBeVisible();
});
```
Keep E2E few and focused on revenue/critical paths; they're slower and flakier. Run in CI against a preview environment.

## CI hook
`npm run lint && npm run typecheck && npm test -- --coverage` → coverage report to SonarQube ([aws/08](../aws/08-cicd-observability-and-cost.md)).

## Exercise
Write: a reducer unit test, a component test of the form with validation errors, an MSW-backed list test, and one Playwright happy-path. Profile the list with 5,000 items, then fix with virtualization and record before/after.

## Interview Q&A
- **How do you find why a component re-renders?** Profiler "Why did this render", `why-did-you-render`, check parent state/context/prop identity.
- **What do you test in a component?** User-visible behavior and contracts (rendered output, interactions, API calls), not internal state.
- **Mocking the network?** MSW at the network boundary, not mocking `fetch` or hooks, so the whole stack runs.
- **How do you reduce a large bundle?** Analyze, split by route, lazy-load heavy libs, drop/replace deps, compress.
- **Flaky tests?** Await UI with `findBy`/`expect.poll`, avoid arbitrary timeouts, isolate data, deterministic time (`vi.useFakeTimers`).
