# TypeScript 04 — Async, Modules, tsconfig and Tooling

## Async/await and Promises

```ts
async function loadDashboard(userId: string) {
  // independent calls in parallel, not sequentially
  const [expenses, summary] = await Promise.all([
    request<Expense[]>(`/expenses?user=${userId}`),
    request<Summary>(`/reports/summary?month=2026-10`),
  ]);
  return { expenses, summary };
}

// don't fail everything because one thing failed
const results = await Promise.allSettled([a(), b()]);
results.forEach(r => r.status === "fulfilled" ? use(r.value) : log(r.reason));

// timeout + cancellation with AbortController
const ctrl = new AbortController();
const t = setTimeout(() => ctrl.abort(), 5000);
try {
  const res = await fetch("/api/expenses", { signal: ctrl.signal });
} finally { clearTimeout(t); }
```

Know: `Promise.all` (fail fast), `allSettled` (all outcomes), `race` (first to settle), `any` (first success). `async` functions always return a `Promise`. `await` in a loop is sequential. Always handle rejections (`try/catch` or `.catch`); unhandled rejections crash Node processes.

### Event loop (they ask!)
Call stack → microtask queue (promise callbacks, `queueMicrotask`) runs fully after each task → macrotask queue (timers, I/O). So `Promise.then` runs before `setTimeout(…, 0)`.

```ts
console.log(1);
setTimeout(() => console.log(2), 0);
Promise.resolve().then(() => console.log(3));
console.log(4);
// 1 4 3 2
```

## Modules
- ES modules: `import { x } from "./x.js"`, `export default` (prefer named exports: easier refactors/tree shaking).
- `import type { Expense } from "./types"` is erased entirely; with `verbatimModuleSyntax` the compiler enforces it.
- **Barrel files** (`index.ts` re-exports) can hurt tree-shaking and circular imports, so use sparingly.
- Node ESM quirk: relative imports need file extensions (`./x.js`) under `"moduleResolution": "NodeNext"`; bundlers (Vite) use `"bundler"`.

## `tsconfig.json` — what each important flag does

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",                    // emitted JS level
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",         // Vite/webpack; "NodeNext" for Node apps
    "strict": true,                        // the big one
    "noUncheckedIndexedAccess": true,      // arr[i] is T | undefined
    "exactOptionalPropertyTypes": true,    // optional ≠ "may be undefined"
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true,               // required by esbuild/SWC/Babel (per-file transpile)
    "skipLibCheck": true,                  // faster; don't type-check .d.ts in node_modules
    "jsx": "react-jsx",                    // no need to import React
    "noEmit": true,                        // bundler emits; tsc only type-checks
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }          // alias; bundler must be configured to match
  },
  "include": ["src"]
}
```

Key insight: with Vite/esbuild, **type-checking and transpiling are separate**. The build doesn't catch type errors, so CI must run `tsc --noEmit`.

## Node + TypeScript (MCP servers, Lambda, CDK)

```bash
npm i -D typescript tsx @types/node
npx tsx src/index.ts          # run TS directly in dev
npx tsc -p . --outDir dist    # build for production
```
For Lambda use esbuild (`sam`/CDK `NodejsFunction` bundle it). Types for AWS: `@types/aws-lambda`.

```ts
import type { APIGatewayProxyEventV2, APIGatewayProxyResultV2 } from "aws-lambda";

export const handler = async (event: APIGatewayProxyEventV2): Promise<APIGatewayProxyResultV2> => {
  return { statusCode: 200, body: JSON.stringify({ ok: true, path: event.rawPath }) };
};
```

## Lint, format, test

```bash
npm i -D eslint typescript-eslint prettier vitest
```
- **ESLint + typescript-eslint** (flat config `eslint.config.js`): `no-floating-promises`, `no-explicit-any`, `consistent-type-imports`.
- **Prettier** for formatting (don't argue style in reviews).
- **Vitest** (Jest-compatible API, Vite-native) for unit tests.

```ts
// money.test.ts
import { describe, it, expect } from "vitest";
import { formatMoney } from "./money";

describe("formatMoney", () => {
  it("formats cents", () => expect(formatMoney(1250, "USD")).toBe("$12.50"));
  it("handles zero", () => expect(formatMoney(0, "USD")).toBe("$0.00"));
});
```

CI gate: `npm run lint && npm run typecheck && npm test`.

## Declaration files and third-party types
- Packages ship types or use `@types/pkg` from DefinitelyTyped.
- Extend env typing: `src/vite-env.d.ts` → `interface ImportMetaEnv { readonly VITE_API_URL: string }`.
- Module augmentation to add a field to a library type; `declare module "x"` for untyped packages.

## Exercise
Create a Node + TS script that fetches `/expenses` with a timeout, validates with Zod, groups by category with the generic `groupBy`, and prints totals. Add a Vitest test and run `tsc --noEmit` in a GitHub Actions job.

## Interview Q&A
- **Why can Vite builds succeed with type errors?** Vite strips types without checking; run `tsc --noEmit`.
- **`Promise.all` vs `allSettled`?** Fail-fast vs collect every outcome.
- **Output order of the snippet above?** `1 4 3 2` — microtasks before timers.
- **`import type`?** Guarantees erasure, avoids runtime import/cycles.
- **What does `isolatedModules` imply?** Each file must be transpilable alone (no `const enum` across files, re-exporting types needs `export type`).
