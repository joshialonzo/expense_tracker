# MCP 04 — Building an MCP Client and Wiring it to Amazon Bedrock

## Why it matters
The JD wants AI features on Bedrock. This is the glue: **an app that connects to MCP servers, hands their tools to a Bedrock model, executes the tool calls the model requests, and loops until the model answers.** If you can explain and code this loop you understand agents.

> **Structure:** the loop below is the "learn it" version. The production structure splits it into a `RunAssistant` use case with `LlmClient` and `ToolGateway` ports, plus Bedrock and MCP adapters, so it's testable with scripted fakes ([07](07-mcp-and-assistant-as-adapters.md)).

## The agent loop

```
1. client.listTools()  → convert each MCP tool to a Bedrock Converse toolSpec
2. Send user message + tools to Bedrock (converse)
3. If stopReason == "tool_use":
      for each toolUse block: client.callTool(name, input)
      append the assistant message + a user message with toolResult blocks
      goto 2
4. Else: the model's text is the final answer
```
(The *host* — your code — runs this loop. The model never talks to the MCP server directly.)

## TypeScript client (connect and call)

```ts
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

const client = new Client({ name: "expense-assistant", version: "1.0.0" });
await client.connect(new StdioClientTransport({ command: "npx", args: ["tsx", "../mcp-expenses/src/stdio.ts"] }));

const { tools } = await client.listTools();
const result = await client.callTool({ name: "monthly_summary", arguments: { month: "2026-10" } });
```
For a remote server use `StreamableHTTPClientTransport` with the server URL and an `Authorization` header (`requestInit: { headers: … }`).

## Python host: MCP client + Bedrock Converse

```python
# agent.py
import asyncio, json, os
import boto3
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

MODEL_ID = os.environ["BEDROCK_MODEL_ID"]            # from config; verify in console
bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")

def to_tool_spec(t) -> dict:
    return {"toolSpec": {"name": t.name, "description": t.description or "", "inputSchema": {"json": t.inputSchema}}}

async def run(question: str, max_turns: int = 6) -> str:
    params = StdioServerParameters(command="python", args=["../mcp-expenses-py/server.py"])
    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            tools = (await session.list_tools()).tools
            tool_config = {"tools": [to_tool_spec(t) for t in tools]}

            messages = [{"role": "user", "content": [{"text": question}]}]
            for _ in range(max_turns):                                   # hard cap: no infinite loops
                resp = bedrock.converse(
                    modelId=MODEL_ID,
                    system=[{"text": "You are an expense assistant. Use tools for facts; never guess numbers."}],
                    messages=messages,
                    toolConfig=tool_config,
                    inferenceConfig={"maxTokens": 1000, "temperature": 0},
                )
                msg = resp["output"]["message"]
                messages.append(msg)

                if resp["stopReason"] != "tool_use":
                    return "".join(b.get("text", "") for b in msg["content"])

                results = []
                for block in msg["content"]:
                    if "toolUse" not in block:
                        continue
                    tu = block["toolUse"]
                    try:
                        out = await session.call_tool(tu["name"], tu["input"])
                        text = "\n".join(c.text for c in out.content if getattr(c, "type", "") == "text")
                        results.append({"toolResult": {"toolUseId": tu["toolUseId"],
                                                       "content": [{"text": text}],
                                                       "status": "error" if out.isError else "success"}})
                    except Exception as e:                                # tool crashed: tell the model
                        results.append({"toolResult": {"toolUseId": tu["toolUseId"],
                                                       "content": [{"text": f"Tool failed: {e}"}],
                                                       "status": "error"}})
                messages.append({"role": "user", "content": results})
            return "Stopped: too many tool turns."

if __name__ == "__main__":
    print(asyncio.run(run("How much did I spend on transport in 2026-10, and what was my top category?")))
```

Details worth saying out loud:
- **Schema translation**: MCP `inputSchema` is JSON Schema → Bedrock `toolSpec.inputSchema.json` is also JSON Schema, so mapping is direct.
- **`toolUseId` must be echoed** in the `toolResult`.
- **Parallel tool calls**: a single assistant message can contain several `toolUse` blocks; execute them (concurrently if independent) and return all results in one user message.
- **Guard rails in the loop**: max turns, token budget, per-tool timeouts, allow-list of tools, user confirmation for destructive tools (`annotations.destructiveHint`).
- **Context management**: truncate/summarize large tool outputs so they don't blow the context window or the bill.

## Putting it behind the web app

```
React (chat UI) ──SSE──► Agent API (FastAPI on ECS) ──► Bedrock (converse_stream)
                                      └──► MCP client ──► MCP servers (expenses, ...)
```
- The Agent API holds the user's JWT, passes it to the MCP server (so tools act *as the user*).
- Stream tokens to the browser with SSE (`StreamingResponse` in FastAPI) from `converse_stream`.
- Persist conversations (DynamoDB) with a TTL; rate-limit per user.

React side (streaming consumption sketch):

```tsx
async function ask(q: string, onToken: (t: string) => void) {
  const res = await fetch("/api/assistant", {
    method: "POST",
    headers: { "Content-Type": "application/json", ...(await authHeader()) },
    body: JSON.stringify({ q }),
  });
  const reader = res.body!.pipeThrough(new TextDecoderStream()).getReader();
  for (;;) {
    const { value, done } = await reader.read();
    if (done) break;
    onToken(value);
  }
}
```

## Bedrock Agents, AgentCore and MCP (what to say)
Amazon Bedrock also offers managed agents and an agent runtime/gateway offering (Bedrock **AgentCore**) that can host agents and expose/consume tools, including MCP-compatible ones. Features and names are evolving, so say: "I'd evaluate managed agent services against a thin custom loop like the above for control, cost and portability; MCP keeps tools portable either way." Also note that desktop hosts like Claude Desktop/IDEs are ready-made MCP clients: you only need to build the server for them.

## Evaluate the agent
Golden questions with expected tool calls and answers; assert on (a) which tools were invoked, (b) arguments, (c) final answer facts. Run in CI on prompt/tool-description changes ([aws/07](../aws/07-bedrock-and-ai-integration.md)).

## Exercise
Run `agent.py` against your Python server. Then: (1) add a second MCP server (e.g. the TS one) and namespace tool names (`expenses__list`), (2) add a confirmation prompt before `delete_expense`, (3) add a token/turn budget, (4) log every tool call with user id and latency.

## Interview Q&A
- **Explain how an LLM uses an MCP tool.** Client lists tools → host passes schemas to the model → model emits a structured tool call → host (client) executes via MCP → result is fed back → model continues.
- **Where is the trust boundary?** Model output is untrusted input to your host; validate arguments (the server does too), confirm destructive actions, scope credentials.
- **How do you bound an agent?** Max turns, budgets, timeouts, allow-lists, idempotent tools, observability.
- **How would you connect many servers?** One client per server; aggregate and namespace tools; route calls back by prefix.
- **What if two tools have similar descriptions?** The model picks wrongly; rewrite descriptions, merge tools, or filter the tool list per task.
