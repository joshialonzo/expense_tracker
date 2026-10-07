# MCP 06 — Interview Questions and Drills

## 60-second answers to rehearse
1. **What is MCP and what problem does it solve?** Open JSON-RPC-based protocol standardizing how AI hosts connect to tools/data; reduces M×N integrations to M+N.
2. **Describe the architecture.** Host → client(s) → server(s); 1 client per server; capability negotiation at `initialize`.
3. **Tools vs resources vs prompts?** Model-controlled actions / application-controlled context / user-controlled templates.
4. **Transports?** stdio (local subprocess) and Streamable HTTP (remote; supersedes HTTP+SSE).
5. **MCP vs function calling vs REST?** MCP packages and serves tools; function calling is the model-level mechanism; REST is an app API that an MCP server may wrap.
6. **What is sampling/roots/elicitation?** Client-offered features: server asks for an LLM completion / boundary hints / user input.
7. **How do you secure a remote MCP server?** OAuth 2.1 resource server pattern, audience-bound JWTs, per-user authz, rate limits, audit logs, WAF.
8. **What is prompt injection and tool poisoning?** See [05](05-security-and-deployment-on-aws.md).
9. **How do you design a good tool?** Task-oriented, strict schema, descriptive, concise output, actionable errors, annotations.
10. **How would you test an MCP server?** In-memory client/server tests, Inspector manually, schema snapshot tests, LLM evals for descriptions.
11. **How do you deploy on AWS?** Fargate+ALB (stateful/streaming) or Lambda (stateless); Cognito for auth; CloudWatch/X-Ray.
12. **How does the agent loop work with Bedrock?** [04](04-clients-and-bedrock-integration.md).

## Whiteboard: "Design an AI assistant for the expense tracker"

```
React chat UI ──► Assistant API (ECS) ──► Bedrock (Converse, tool use, Guardrails)
                        │
                        ├─► MCP client ─► Expense MCP server (reads/writes via service layer, user-scoped)
                        ├─► MCP client ─► Receipts MCP server (S3 + Textract)
                        └─► DynamoDB (conversation history, TTL)
Cognito JWT flows UI → API → MCP; every tool authorizes by `sub`.
```
Cover: identity propagation, tool design, destructive-action confirmation, streaming, rate limiting & token budgets, observability, eval strategy, cost (model routing, caching), failure modes (tool timeouts, model errors, partial results).

## Code drills
1. **Write a tool** `search_expenses(query, from, to)` with Zod/Pydantic validation; return at most 20 rows with a `next_cursor`.
2. **Convert MCP tools to Bedrock toolSpecs** (10 lines).
3. **Write the agent loop** with max turns and per-tool timeout.
4. **Add auth middleware** that validates a JWT and injects `userId` into the tool context.
5. **Write an in-memory test** that proves user A can't see user B's data.

## Trap questions
- *"Can the MCP server force the model to call a tool?"* No. The host/model decide; servers only advertise.
- *"Is MCP an agent framework?"* No, it's a connectivity protocol. Orchestration (loops, planning) lives in the host.
- *"Should every REST endpoint become a tool?"* No. Curate for tasks; too many overlapping tools degrade tool selection and cost tokens (each tool schema consumes context).
- *"Why not just give the model a SQL tool?"* Injection and over-privilege; prefer narrow parameterized tools, or a read-only role with strict row-level security if you must.
- *"What if the tool returns 5 MB?"* Paginate/summarize, enforce limits; context windows and cost.

## Say-it-in-one-breath summary
"MCP standardizes how an AI host discovers and calls tools and fetches context from servers over JSON-RPC, via stdio locally or Streamable HTTP remotely. I build small, schema-strict, user-scoped tools, authenticate remote servers with OAuth-issued JWTs, run them on Fargate or Lambda, and assume the model and tool output are untrusted, so destructive actions are confirmed and everything is logged."

## Questions to ask the interviewer about MCP/AI
- Which AI use cases are in production vs experimental?
- Are you building MCP servers, consuming third-party ones, or both?
- How do you evaluate prompt/tool changes before release?
- How do you handle model/provider changes across Azure AI Foundry and Bedrock?
