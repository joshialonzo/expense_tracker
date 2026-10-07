# MCP 01 — What is the Model Context Protocol?

## Why it matters
MCP is a "must have" topic and it's the newest on the list, so interviewers probe whether you understand it beyond buzzwords: the problem, the architecture, the primitives, transports, and security.

## The problem
Before MCP, connecting *M* AI apps to *N* tools/data sources meant **M × N custom integrations** (each app's own plugin format). MCP is an **open standard** (introduced by Anthropic, now broadly adopted) that turns it into **M + N**: any MCP **client** can talk to any MCP **server**. Analogy: USB-C for AI integrations, or LSP (Language Server Protocol) for AI context and tools.

## Architecture

```
┌──────────────── Host (Claude Desktop, VS Code, your app) ───────────────┐
│   LLM  ◄──►  MCP Client 1 ──────────────► MCP Server A (expenses)       │
│              MCP Client 2 ──────────────► MCP Server B (github)         │
└─────────────────────────────────────────────────────────────────────────┘
```
- **Host**: the AI application the user interacts with. It owns the LLM, the UI, and consent/permission decisions.
- **Client**: a connector inside the host; one client ↔ one server (1:1 session).
- **Server**: a program exposing capabilities to clients (local process or remote service).
- Messages are **JSON-RPC 2.0** (requests, responses, notifications). Sessions start with an **initialize** handshake that negotiates protocol version and **capabilities**.

## Server primitives (what servers expose)

| Primitive | Controlled by | What | Expense-tracker example |
|---|---|---|---|
| **Tools** | The *model* (model decides to call; host usually asks the user to approve) | Functions with JSON-Schema inputs that do things or compute | `add_expense`, `list_expenses`, `monthly_summary` |
| **Resources** | The *application* (host/user attaches as context) | Read-only data addressed by URI | `expenses://2026-10/summary`, `receipt://abc.png` |
| **Prompts** | The *user* (explicitly invoked, e.g. slash command) | Reusable prompt templates with arguments | `/review-budget month=2026-10` |

## Client primitives (what clients offer servers)
- **Sampling**: server asks the host's LLM to generate a completion (server needs no API key; host stays in control and can require approval).
- **Roots**: client tells the server which filesystem/URI boundaries it may operate in.
- **Elicitation**: server asks the user for more information mid-task (via the client).

## Transports

| Transport | Use | Notes |
|---|---|---|
| **stdio** | Local servers launched as a subprocess by the host | stdin/stdout JSON-RPC; **never write logs to stdout**, use stderr |
| **Streamable HTTP** | Remote/hosted servers | Single HTTP endpoint (POST, optional SSE streaming for server→client messages); supersedes the older HTTP+SSE transport; supports stateless and session modes |

For AWS: run your remote MCP server on **ECS Fargate** or **Lambda behind API Gateway/Function URL** (stateless mode fits Lambda) ([05](05-security-and-deployment-on-aws.md)).

## Lifecycle

```
client → initialize {protocolVersion, capabilities, clientInfo}
server → result     {protocolVersion, capabilities: {tools:{}, resources:{}, prompts:{}}, serverInfo}
client → notifications/initialized
client → tools/list            → server returns tool definitions (name, description, inputSchema)
model decides → client → tools/call {name, arguments} → server returns {content:[…], isError?}
server → notifications/tools/list_changed   (if the tool set changes)
```

A tool call on the wire:

```json
{"jsonrpc":"2.0","id":7,"method":"tools/call",
 "params":{"name":"add_expense","arguments":{"amountCents":1250,"category":"food"}}}

{"jsonrpc":"2.0","id":7,"result":{
  "content":[{"type":"text","text":"Created expense e_123 for $12.50"}],
  "isError":false}}
```
Tool *execution failures* are returned as results with `isError: true` (so the model can see and recover); protocol errors (unknown method, bad params) are JSON-RPC errors.

## MCP vs other things (expect comparison questions)

| | MCP | Plain function/tool calling | REST/OpenAPI | RAG |
|---|---|---|---|---|
| What | Open protocol connecting hosts to servers (discovery + invocation + context) | Model-API feature: model emits a structured call, *your code* runs it | API contract for programs | Retrieve text into prompt |
| Discovery | Dynamic `tools/list` | Static in your request | Spec file | n/a |
| Relationship | MCP **builds on** tool calling: client converts MCP tools into the model's tool format | Underlying mechanism | Often wrapped *by* an MCP server | An MCP *resource/tool* can provide retrieval |

Key line: **"Tool calling is how a model asks; MCP is how tools are packaged, discovered and served in a standard way."**

## When to build an MCP server
- You want multiple AI apps/agents (Claude Desktop, IDE assistants, your Bedrock agent) to use the same capability.
- You want a standard, discoverable interface with consent and auditability.
- Not needed when only your own app calls a couple of internal functions: plain tool calling is simpler.

## Design principles for good tools
1. **Small, task-oriented tools** (not a 1:1 mirror of every REST endpoint; fewer, clearer tools help the model pick correctly).
2. **Descriptions are prompts**: say what it does, when to use it, input formats, and what it returns.
3. **Strict schemas** (enums, min/max, formats) and **validation** server-side.
4. **Return concise, structured, useful results** (don't dump 10k rows into the context window; paginate/summarize).
5. **Helpful errors** the model can act on ("category must be one of …").
6. **Annotate**: `readOnlyHint`, `destructiveHint`, `idempotentHint` so hosts can decide on confirmations.
7. **Least privilege & identity-scoped**: act as the calling user, never as an all-powerful service account.

## Spec/ecosystem pointers
Specification and docs live at modelcontextprotocol.io; official SDKs for TypeScript, Python, Java, Kotlin, C#, Go, and more. The protocol evolves (versions are date-stamped, e.g. `2025-06-18`), so say "I'd check the current spec revision" for details like auth and transports.

## Exercise
Install the MCP Inspector (`npx @modelcontextprotocol/inspector`) and connect it to any public example server. Capture the `initialize`, `tools/list` and `tools/call` messages and annotate each field.

## Interview Q&A
- **What problem does MCP solve?** M×N integration explosion → standard client/server protocol.
- **Tools vs resources vs prompts?** Model-controlled actions vs app-controlled context vs user-controlled templates.
- **stdio vs Streamable HTTP?** Local subprocess vs remote networked service (auth, scaling, multi-client).
- **Who decides whether a tool runs?** The host, which should keep a human in the loop for sensitive actions. The server cannot force the model.
- **How is MCP different from function calling?** See table above.
- **What is sampling?** Server requests an LLM completion through the client.
