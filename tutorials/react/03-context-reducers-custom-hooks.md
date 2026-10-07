# React 03 — Context, useReducer, Custom Hooks and State Management

## Why it matters
"How do you manage state in a large app?" is a staple. Have a decision framework, not a library name.

## State taxonomy (use this framework in the interview)

| Kind | Examples | Tool |
|---|---|---|
| **Local UI state** | open modal, input text | `useState` |
| **Shared UI state** | theme, current user, sidebar | Context, or Zustand |
| **Complex local logic** | form wizard, cart | `useReducer` |
| **Server state** (cached remote data) | expenses, reports | **TanStack Query** ([04](04-data-fetching-tanstack-query.md)) |
| **URL state** | filters, page, selected id | router search params |
| **Form state** | validation, dirty, errors | React Hook Form + Zod |

Most "global state" problems vanish when server state lives in a query cache and filters live in the URL.

## useReducer

Use when next state depends on several fields or when transitions have names.

```tsx
type State = { items: Expense[]; filter: Category | "all"; error?: string };
type Action =
  | { type: "loaded"; items: Expense[] }
  | { type: "added"; item: Expense }
  | { type: "removed"; id: string }
  | { type: "filterChanged"; filter: State["filter"] }
  | { type: "failed"; error: string };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "loaded":        return { ...state, items: action.items, error: undefined };
    case "added":         return { ...state, items: [action.item, ...state.items] };
    case "removed":       return { ...state, items: state.items.filter(e => e.id !== action.id) };
    case "filterChanged": return { ...state, filter: action.filter };
    case "failed":        return { ...state, error: action.error };
  }
}

const [state, dispatch] = useReducer(reducer, { items: [], filter: "all" });
dispatch({ type: "removed", id: "42" });
```
Reducers are pure and trivially unit-testable. `dispatch` identity is stable, so it's safe to pass down.

## Context

Solves **prop drilling** for values many components need. It is *dependency injection*, not a state manager: every consumer re-renders when the value changes.

```tsx
type AuthValue = { user: User | null; signIn: () => Promise<void>; signOut: () => Promise<void> };
const AuthContext = createContext<AuthValue | null>(null);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  const signIn  = useCallback(async () => { setUser(await auth.signIn()); }, []);
  const signOut = useCallback(async () => { await auth.signOut(); setUser(null); }, []);

  // memoize the value so consumers don't re-render on unrelated parent renders
  const value = useMemo(() => ({ user, signIn, signOut }), [user, signIn, signOut]);
  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error("useAuth must be used within AuthProvider");
  return ctx;
}
```
(React 19 also allows `<AuthContext value={…}>` without `.Provider`.)

Performance guidance: split contexts by update frequency (e.g. `ThemeContext` vs `UserContext`), keep the value stable, or use a store with selectors (Zustand) for frequently changing shared state.

### Zustand (lightweight external store)

```ts
import { create } from "zustand";

type UiStore = { sidebarOpen: boolean; toggle: () => void };
export const useUi = create<UiStore>((set) => ({
  sidebarOpen: true,
  toggle: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
}));

const open = useUi((s) => s.sidebarOpen);    // selector → re-render only when this slice changes
```
Redux Toolkit is the established alternative (big teams, devtools, middleware). Mention: Context for DI/low-churn values, Zustand/RTK for frequent global client state.

## Custom hooks

A custom hook is just a function starting with `use` that calls other hooks. It shares **logic**, not state: each call gets its own state.

```tsx
export function useToggle(initial = false) {
  const [on, setOn] = useState(initial);
  const toggle = useCallback(() => setOn(v => !v), []);
  return { on, toggle, set: setOn } as const;
}

export function useOnlineStatus() {
  return useSyncExternalStore(
    (cb) => {
      window.addEventListener("online", cb);
      window.addEventListener("offline", cb);
      return () => { window.removeEventListener("online", cb); window.removeEventListener("offline", cb); };
    },
    () => navigator.onLine,
    () => true
  );
}
```
`useSyncExternalStore` is the correct primitive for subscribing to external stores (browser APIs, Zustand uses it).

## Error boundaries
Class-based (or `react-error-boundary` library) components that catch render errors in children and show a fallback. They don't catch errors in event handlers or async code.

```tsx
import { ErrorBoundary } from "react-error-boundary";
<ErrorBoundary fallback={<p>Something went wrong.</p>} onError={reportToSentryOrCloudWatch}>
  <Dashboard />
</ErrorBoundary>
```

## Suspense and lazy loading

```tsx
const Reports = lazy(() => import("./Reports"));
<Suspense fallback={<Spinner />}><Reports /></Suspense>
```
Route-level code splitting is the cheapest big bundle win.

## Exercise
Implement an `AuthProvider` + `useAuth`, a `ThemeProvider`, and an expenses reducer with unit tests (Vitest) for each action. Then argue: which of these would you move to Zustand and why?

## Interview Q&A
- **Context vs Redux?** Context transports values (and re-renders all consumers on change); Redux/Zustand manage updates with selectors, middleware and devtools.
- **How do you avoid unnecessary re-renders with context?** Split contexts, memoize the value, or move to a selector-based store.
- **`useReducer` vs `useState`?** Related fields/complex transitions → reducer.
- **What can't an error boundary catch?** Event handlers, async code, SSR errors, errors in the boundary itself.
- **Custom hook shares state?** No, logic only; use context/store to share state.
