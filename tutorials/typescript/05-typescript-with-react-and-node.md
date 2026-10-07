# TypeScript 05 — TypeScript in React and Node/Express APIs

## Why it matters
Interviewers often combine topics: "type this component", "type this hook", "type this Express handler". Practice the idioms.

## React + TypeScript

### Props and children

```tsx
import type { ReactNode } from "react";

type ExpenseRowProps = {
  expense: Expense;
  onDelete: (id: string) => void;
  actions?: ReactNode;
};

export function ExpenseRow({ expense, onDelete, actions }: ExpenseRowProps) {
  return (
    <li>
      {expense.description ?? expense.category} — {formatMoney(expense.amountCents, expense.currency)}
      <button onClick={() => onDelete(expense.id)}>Delete</button>
      {actions}
    </li>
  );
}
```
Prefer plain function components with a typed props object over `React.FC` (no implicit children, simpler generics).

### Events and refs

```tsx
function SearchBox({ onSearch }: { onSearch: (q: string) => void }) {
  const inputRef = useRef<HTMLInputElement>(null);        // null initially
  const [q, setQ] = useState("");                         // inferred string
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => setQ(e.target.value);
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    onSearch(q);
    inputRef.current?.focus();
  };
  return (
    <form onSubmit={handleSubmit}>
      <input ref={inputRef} value={q} onChange={handleChange} />
    </form>
  );
}
```

### `useState` with a union / nullable

```tsx
const [selected, setSelected] = useState<Expense | null>(null);
const [state, setState] = useState<RemoteData<Expense[]>>({ status: "idle" });
```

### Generic components

```tsx
type SelectProps<T extends string> = {
  value: T;
  options: readonly T[];
  onChange: (v: T) => void;
  label: string;
};

export function Select<T extends string>({ value, options, onChange, label }: SelectProps<T>) {
  return (
    <label>
      {label}
      <select value={value} onChange={(e) => onChange(e.target.value as T)}>
        {options.map((o) => <option key={o} value={o}>{o}</option>)}
      </select>
    </label>
  );
}

<Select label="Category" value={cat} options={CATEGORIES} onChange={setCat} />   // T inferred as Category
```

### Typed custom hook

```tsx
export function useLocalStorage<T>(key: string, initial: T) {
  const [value, setValue] = useState<T>(() => {
    try {
      const raw = localStorage.getItem(key);
      return raw ? (JSON.parse(raw) as T) : initial;
    } catch { return initial; }
  });
  useEffect(() => { localStorage.setItem(key, JSON.stringify(value)); }, [key, value]);
  return [value, setValue] as const;     // tuple, not (T | Dispatch)[]
}
```
`as const` preserves the tuple type. Returning an object is often cleaner than tuples for >2 values.

### Context with a safe default

```tsx
const AuthContext = createContext<AuthValue | null>(null);

export function useAuth(): AuthValue {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error("useAuth must be used inside <AuthProvider>");
  return ctx;
}
```

### Reuse native element props

```tsx
type ButtonProps = React.ComponentPropsWithoutRef<"button"> & { variant?: "primary" | "ghost" };
export function Button({ variant = "primary", className, ...rest }: ButtonProps) {
  return <button className={`btn btn-${variant} ${className ?? ""}`} {...rest} />;
}
```

## Node / Express with TypeScript (also the base for MCP servers)

```ts
import express, { type Request, type Response, type NextFunction } from "express";
import { z } from "zod";

const app = express();
app.use(express.json());

const CreateExpense = z.object({
  amountCents: z.number().int().positive(),
  currency: z.enum(["USD", "EUR", "MXN"]),
  category: z.string().min(1),
  description: z.string().max(200).optional(),
});

declare global {
  namespace Express { interface Request { userId?: string } }
}

function auth(req: Request, res: Response, next: NextFunction) {
  const token = req.headers.authorization?.replace(/^Bearer /, "");
  if (!token) return res.status(401).json({ code: "unauthorized" });
  req.userId = verify(token).sub;      // verify() = your JWT check
  next();
}

app.post("/expenses", auth, async (req, res) => {
  const parsed = CreateExpense.safeParse(req.body);
  if (!parsed.success) return res.status(422).json({ code: "validation_error", issues: parsed.error.flatten() });
  const created = await repo.create({ ...parsed.data, userId: req.userId! });
  res.status(201).location(`/expenses/${created.id}`).json(created);
});

app.use((err: unknown, _req: Request, res: Response, _next: NextFunction) => {
  console.error(err);
  res.status(500).json({ code: "internal_error" });
});
```

**Share types front↔back**: put Zod schemas/DTOs in a `packages/shared` workspace (npm/pnpm workspaces) and import in both, so the contract is compiled, not hoped. Or generate types from OpenAPI (`openapi-typescript`) when the backend is Python/Go: the contract is the OpenAPI spec (FastAPI/swag generate it).

## Exercise
Build `<ExpenseForm>` with typed props, a generic `<Select<Category>>`, and `useLocalStorage` for a draft. Generate TS types from the FastAPI OpenAPI: `npx openapi-typescript http://localhost:8000/openapi.json -o src/api/schema.d.ts`.

## Interview Q&A
- **Why not `React.FC`?** Historically implied `children`; plain function with typed props is simpler and generics work well.
- **How do you type a `useRef` for a DOM node vs a mutable value?** `useRef<HTMLInputElement>(null)` for DOM (readonly `.current`), `useRef<number>(0)` for mutable instance value.
- **How do you keep FE/BE types in sync with a Python/Go backend?** OpenAPI → generated client/types, contract tests.
- **Event types?** `React.ChangeEvent<HTMLInputElement>`, `React.MouseEvent<HTMLButtonElement>`, `React.FormEvent<HTMLFormElement>`.
