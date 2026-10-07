# MCP 02 — Build an Expense MCP Server in TypeScript

## Goal
An MCP server exposing the expense tracker: tools (`add_expense`, `list_expenses`, `monthly_summary`), a resource (`expenses://{month}/summary`) and a prompt (`budget_review`). First over **stdio** (for Claude Desktop/Inspector), then **Streamable HTTP** (remote).

> The SDK evolves quickly. The shape below matches the official `@modelcontextprotocol/sdk` (v1.x) high-level `McpServer` API. If an import path or method name differs in your installed version, check the SDK README.

## Setup

```bash
mkdir mcp-expenses && cd mcp-expenses
npm init -y && npm pkg set type=module
npm i @modelcontextprotocol/sdk zod
npm i -D typescript tsx @types/node
npx tsc --init --target ES2022 --module NodeNext --moduleResolution NodeNext --strict --outDir dist
```

## A tiny data layer (swap for the real API later)

```ts
// src/store.ts
export type Expense = { id: string; amountCents: number; currency: string; category: string; description?: string; date: string };

const expenses: Expense[] = [
  { id: "e1", amountCents: 1250, currency: "USD", category: "food", description: "Lunch", date: "2026-10-01" },
  { id: "e2", amountCents: 4500, currency: "USD", category: "transport", description: "Train pass", date: "2026-10-03" },
];

export const store = {
  list: (month?: string, category?: string) =>
    expenses.filter(e => (!month || e.date.startsWith(month)) && (!category || e.category === category)),
  add: (e: Omit<Expense, "id">): Expense => {
    const created = { id: `e${expenses.length + 1}`, ...e };
    expenses.push(created);
    return created;
  },
};
```

## The server

> **Structure tip:** this tutorial keeps tool bodies inline for readability. For anything beyond a demo, move rules into an `ExpenseTools` class behind an `ExpensesGateway` port and keep `registerTool` calls as thin adapters ([07](07-mcp-and-assistant-as-adapters.md)).

```ts
// src/server.ts
import { McpServer, ResourceTemplate } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";
import { store } from "./store.js";

const Month = z.string().regex(/^\d{4}-\d{2}$/, "use YYYY-MM");
const money = (c: number, cur = "USD") =>
  new Intl.NumberFormat("en-US", { style: "currency", currency: cur }).format(c / 100);

export function buildServer() {
  const server = new McpServer({ name: "expense-tracker", version: "1.0.0" });

  // ---------- TOOLS ----------
  server.registerTool(
    "add_expense",
    {
      title: "Add expense",
      description:
        "Record a new expense. Use when the user says they spent money. Amount is in the smallest currency unit (cents), e.g. $12.50 = 1250.",
      inputSchema: {
        amountCents: z.number().int().positive().describe("Amount in cents"),
        category: z.enum(["food", "transport", "housing", "fun", "other"]),
        description: z.string().max(200).optional(),
        date: z.string().date().describe("ISO date YYYY-MM-DD"),
      },
      annotations: { readOnlyHint: false, destructiveHint: false, idempotentHint: false },
    },
    async ({ amountCents, category, description, date }) => {
      const e = store.add({ amountCents, currency: "USD", category, description, date });
      return { content: [{ type: "text", text: `Created ${e.id}: ${money(e.amountCents)} on ${e.category} (${e.date})` }] };
    },
  );

  server.registerTool(
    "list_expenses",
    {
      title: "List expenses",
      description: "List expenses, optionally filtered by month (YYYY-MM) and category. Returns at most `limit` rows, newest first.",
      inputSchema: { month: Month.optional(), category: z.string().optional(), limit: z.number().int().min(1).max(100).default(20) },
      annotations: { readOnlyHint: true },
    },
    async ({ month, category, limit }) => {
      const rows = store.list(month, category).sort((a, b) => b.date.localeCompare(a.date)).slice(0, limit);
      return {
        content: [{ type: "text", text: JSON.stringify(rows) }],
        structuredContent: { expenses: rows },        // machine-readable (supported in newer spec revisions)
      };
    },
  );

  server.registerTool(
    "monthly_summary",
    {
      title: "Monthly summary",
      description: "Total spending per category for a month.",
      inputSchema: { month: Month },
      annotations: { readOnlyHint: true },
    },
    async ({ month }) => {
      const rows = store.list(month);
      if (rows.length === 0) {
        // Execution problem the model can recover from → isError, not a protocol error
        return { isError: true, content: [{ type: "text", text: `No expenses found for ${month}. Try another month.` }] };
      }
      const totals: Record<string, number> = {};
      for (const e of rows) totals[e.category] = (totals[e.category] ?? 0) + e.amountCents;
      const text = Object.entries(totals).sort((a, b) => b[1] - a[1]).map(([c, t]) => `${c}: ${money(t)}`).join("\n");
      return { content: [{ type: "text", text }] };
    },
  );

  // ---------- RESOURCE (templated URI) ----------
  server.registerResource(
    "month-summary",
    new ResourceTemplate("expenses://{month}/summary", { list: undefined }),
    { title: "Monthly expense data", description: "All expenses for a month as JSON", mimeType: "application/json" },
    async (uri, { month }) => ({
      contents: [{ uri: uri.href, mimeType: "application/json", text: JSON.stringify(store.list(String(month))) }],
    }),
  );

  // ---------- PROMPT ----------
  server.registerPrompt(
    "budget_review",
    {
      title: "Budget review",
      description: "Ask the assistant to review a month's spending",
      argsSchema: { month: Month },
    },
    ({ month }) => ({
      messages: [{
        role: "user",
        content: { type: "text", text: `Use the monthly_summary and list_expenses tools for ${month}. Identify the top 3 categories, flag anything unusual, and suggest one concrete saving.` },
      }],
    }),
  );

  return server;
}
```

## Transport 1: stdio (local)

```ts
// src/stdio.ts
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { buildServer } from "./server.js";

const server = buildServer();
await server.connect(new StdioServerTransport());
console.error("expense-tracker MCP server running on stdio");   // stderr only! stdout is the protocol channel
```

Test with the Inspector:

```bash
npx @modelcontextprotocol/inspector npx tsx src/stdio.ts
```
Call each tool in the UI; try invalid input (`month: "October"`) and watch Zod reject it.

Register in Claude Desktop (`claude_desktop_config.json`) or Claude Code:

```json
{ "mcpServers": { "expenses": { "command": "npx", "args": ["tsx", "/abs/path/mcp-expenses/src/stdio.ts"] } } }
```
```bash
claude mcp add expenses -- npx tsx /abs/path/mcp-expenses/src/stdio.ts
```

## Transport 2: Streamable HTTP (remote)

Stateless mode (a new server/transport per request) is simple and scales horizontally, so it suits Lambda/Fargate behind a load balancer.

```ts
// src/http.ts
import express from "express";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { buildServer } from "./server.js";

const app = express();
app.use(express.json());

app.post("/mcp", async (req, res) => {
  const server = buildServer();
  const transport = new StreamableHTTPServerTransport({ sessionIdGenerator: undefined });  // stateless
  res.on("close", () => { transport.close(); server.close(); });
  await server.connect(transport);
  await transport.handleRequest(req, res, req.body);
});

// stateless servers don't support server-initiated streams
app.get("/mcp", (_req, res) => res.status(405).json({ jsonrpc: "2.0", error: { code: -32000, message: "Method not allowed" }, id: null }));

app.get("/health", (_req, res) => res.json({ ok: true }));
app.listen(3000, () => console.error("MCP HTTP on :3000/mcp"));
```
Need server→client features (sampling, progress streams, resumability)? Use **stateful** mode with `sessionIdGenerator: () => randomUUID()` and keep a transport per `Mcp-Session-Id` (needs sticky routing or shared session store when scaled out).

Test: `npx @modelcontextprotocol/inspector` → choose *Streamable HTTP* → `http://localhost:3000/mcp`.

## Calling your real API instead of in-memory data
Replace `store` with calls to the expense REST API. **Pass the end user's identity** through (see [05](05-security-and-deployment-on-aws.md)): for HTTP servers, read the `Authorization` bearer from the incoming request and forward/validate it; never use one shared super-token.

## Testing the server programmatically

```ts
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { InMemoryTransport } from "@modelcontextprotocol/sdk/inMemory.js";
import { buildServer } from "../src/server.js";
import { it, expect } from "vitest";

it("summarizes a month", async () => {
  const [clientT, serverT] = InMemoryTransport.createLinkedPair();
  await buildServer().connect(serverT);
  const client = new Client({ name: "test", version: "0.0.0" });
  await client.connect(clientT);

  const res = await client.callTool({ name: "monthly_summary", arguments: { month: "2026-10" } });
  expect(res.isError).toBeFalsy();
  expect(JSON.stringify(res.content)).toContain("transport");
});
```

## Exercise
Add `delete_expense` (destructive: set `destructiveHint: true`, and require a `confirm: true` argument), a `categories` resource, pagination via `cursor` in `list_expenses`, and tests for each.

## Interview Q&A
- **Why not `console.log` in a stdio server?** stdout carries JSON-RPC frames; stray output corrupts the stream.
- **Where do tool descriptions matter?** They are what the model reads to decide when/how to call: write them like a prompt.
- **Stateless vs stateful HTTP?** Stateless scales trivially (Lambda-friendly) but can't push server-initiated messages or keep session context; stateful needs session affinity/storage.
- **How do you return an error the model can handle?** `isError: true` with an actionable message.
- **How do you validate input?** Zod schema → JSON Schema for the client + runtime validation in the SDK.
