# React 01 — Components, JSX, Props and State

## Why it matters
Core of a must-have topic. Be able to explain *how React thinks*: UI = f(state), re-render on state change, reconcile via the virtual DOM.

## Project setup (Vite + React + TS)

```bash
npm create vite@latest web -- --template react-ts
cd web && npm i && npm run dev
```
(Create React App is deprecated; Vite or a framework like Next.js/React Router are the norm.)

> **Where this goes later:** components stay thin. Validation and money logic move to a framework-free `domain/`, and data access goes behind ports ([08](08-clean-architecture-frontend.md)). Learn the React mechanics here first.

## Mental model
1. **Component** = function that takes `props` and returns UI (JSX).
2. **State** = data that changes over time; changing it schedules a re-render.
3. **Render must be pure**: same props/state → same JSX; no side effects, no mutations.
4. React compares (reconciles) the new element tree with the previous one and updates only the DOM that changed. **Keys** identify list items across renders.
5. Data flows **down** (props), events flow **up** (callbacks).

## JSX rules
- One root (or `<>…</>` fragment). `className`, `htmlFor`. Expressions in `{}`. Conditionals via `&&`/ternary (watch `0 && <X/>` rendering `0`).
- JSX auto-escapes strings → XSS protection (except `dangerouslySetInnerHTML`).

## A first feature: the expense list

```tsx
// src/features/expenses/ExpenseList.tsx
import { useState } from "react";
import type { Expense } from "./types";

type Props = {
  initial: Expense[];
};

export function ExpenseList({ initial }: Props) {
  const [expenses, setExpenses] = useState(initial);
  const [filter, setFilter] = useState("");

  const visible = expenses.filter((e) =>
    e.description?.toLowerCase().includes(filter.toLowerCase()) ?? false
  );
  const totalCents = visible.reduce((sum, e) => sum + e.amountCents, 0);   // derived, NOT state

  function remove(id: string) {
    setExpenses((prev) => prev.filter((e) => e.id !== id));   // immutable update
  }

  return (
    <section>
      <input
        placeholder="Search…"
        value={filter}
        onChange={(e) => setFilter(e.target.value)}
      />
      {visible.length === 0 ? (
        <p>No expenses.</p>
      ) : (
        <ul>
          {visible.map((e) => (
            <ExpenseRow key={e.id} expense={e} onDelete={remove} />
          ))}
        </ul>
      )}
      <strong>Total: {(totalCents / 100).toFixed(2)}</strong>
    </section>
  );
}
```

Key lessons embedded:
- **Derived data isn't state** (`visible`, `totalCents`): computing it avoids sync bugs.
- **Never mutate state** (`expenses.push(...)`); create new arrays/objects (`map`, `filter`, spread).
- **Functional updates** (`setX(prev => …)`) when the next state depends on the previous.
- **`key` must be stable and unique** (an id), never the array index for lists that reorder/delete.

## Controlled vs uncontrolled inputs
- **Controlled**: `value` + `onChange` from state; React is the source of truth (validation, formatting).
- **Uncontrolled**: DOM holds the value, read via ref/`FormData`; fewer re-renders, fine for simple forms.

```tsx
function AddExpense({ onAdd }: { onAdd: (e: Omit<Expense, "id" | "createdAt">) => void }) {
  function handleSubmit(ev: React.FormEvent<HTMLFormElement>) {
    ev.preventDefault();
    const fd = new FormData(ev.currentTarget);
    onAdd({
      amountCents: Math.round(Number(fd.get("amount")) * 100),
      currency: "USD",
      category: String(fd.get("category")),
      description: String(fd.get("description") ?? ""),
    });
    ev.currentTarget.reset();
  }
  return (
    <form onSubmit={handleSubmit}>
      <input name="amount" type="number" step="0.01" min="0.01" required />
      <input name="category" required />
      <input name="description" />
      <button type="submit">Add</button>
    </form>
  );
}
```

## Lifting state up & composition
When two siblings need the same state, move it to their nearest common parent and pass down. Prefer **composition** (`children`, slots as props) over inheritance and over prop-drilling through many layers (then use context, see [03](03-context-reducers-custom-hooks.md)).

```tsx
function Card({ title, children }: { title: string; children: React.ReactNode }) {
  return <div className="card"><h3>{title}</h3>{children}</div>;
}
```

## State batching and snapshots (classic gotcha)

```tsx
const [count, setCount] = useState(0);
function bad()  { setCount(count + 1); setCount(count + 1); }          // +1 (both see the same snapshot)
function good() { setCount(c => c + 1); setCount(c => c + 1); }        // +2
```
State is a *snapshot* per render. `setState` doesn't change the variable in the current closure; it schedules the next render. React batches updates in event handlers (and everywhere since React 18).

## Exercise
Build the list above with: add, delete, category filter (a `<select>`), total. Then extract `ExpenseRow` and `ExpenseFilters` components and lift state appropriately. Do it without `useEffect`; if you reached for it, you probably wanted derived data.

## Interview Q&A
- **What is the virtual DOM / reconciliation?** In-memory tree of elements; React diffs old vs new trees (same type at same position → update, different type → remount; keys match list items) and commits minimal DOM changes.
- **Why are keys needed?** Identity across renders; wrong keys cause state to attach to the wrong item.
- **Props vs state?** Props are inputs from the parent (read-only); state is owned and changed by the component.
- **Why not mutate state?** React uses reference/`Object.is` comparisons to decide to re-render and for memoization; mutations are invisible and break purity.
- **Controlled vs uncontrolled?** See above.
- **What triggers a re-render?** State change, parent re-render (props identity not required), context value change.
