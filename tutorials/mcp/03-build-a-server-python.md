# MCP 03 — Build an Expense MCP Server in Python (FastMCP)

## Why both languages
The JD says Python or Go on the backend, and TypeScript on the frontend. A Python MCP server can reuse the FastAPI domain code ([../fastapi/](../fastapi/)). Interviewers may ask you to build one in whichever language you pick, so know both.

## Setup

> **Pin `mcp<2`.** `mcp` 2.x renamed `FastMCP` to `MCPServer` (`mcp.server.mcpserver`) and changed other APIs. These tutorials and the code in `mcp-server/` and `assistant/` use the v1 API (verified with 1.30). Moving to v2 is a deliberate migration (see the SDK migration guide), not an accidental `pip install -U`.

```bash
mkdir mcp-expenses-py && cd mcp-expenses-py
python -m venv .venv && source .venv/bin/activate
pip install "mcp[cli]<2"       # official Python SDK (includes FastMCP) and the `mcp` dev CLI
```

## The server

```python
# server.py
from datetime import date
from typing import Annotated, Literal
from pydantic import Field
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("expense-tracker")

Category = Literal["food", "transport", "housing", "fun", "other"]

EXPENSES: list[dict] = [
    {"id": "e1", "amount_cents": 1250, "category": "food", "description": "Lunch", "date": "2026-10-01"},
    {"id": "e2", "amount_cents": 4500, "category": "transport", "description": "Train pass", "date": "2026-10-03"},
]

def money(cents: int) -> str:
    return f"${cents / 100:,.2f}"


@mcp.tool()
def add_expense(
    amount_cents: Annotated[int, Field(gt=0, description="Amount in cents, e.g. $12.50 = 1250")],
    category: Category,
    on: Annotated[date, Field(description="Date of the expense (YYYY-MM-DD)")],
    description: str | None = None,
) -> str:
    """Record a new expense. Use when the user says they spent money."""
    e = {"id": f"e{len(EXPENSES) + 1}", "amount_cents": amount_cents, "category": category,
         "description": description, "date": on.isoformat()}
    EXPENSES.append(e)
    return f"Created {e['id']}: {money(amount_cents)} on {category} ({e['date']})"


@mcp.tool()
def list_expenses(
    month: Annotated[str | None, Field(pattern=r"^\d{4}-\d{2}$", description="YYYY-MM")] = None,
    category: Category | None = None,
    limit: Annotated[int, Field(ge=1, le=100)] = 20,
) -> list[dict]:
    """List expenses newest first, optionally filtered by month and category."""
    rows = [e for e in EXPENSES
            if (month is None or e["date"].startswith(month)) and (category is None or e["category"] == category)]
    return sorted(rows, key=lambda e: e["date"], reverse=True)[:limit]


@mcp.tool()
def monthly_summary(month: Annotated[str, Field(pattern=r"^\d{4}-\d{2}$")]) -> str:
    """Total spending per category for a month (YYYY-MM)."""
    rows = [e for e in EXPENSES if e["date"].startswith(month)]
    if not rows:
        raise ValueError(f"No expenses found for {month}. Try another month.")   # surfaced as isError=true
    totals: dict[str, int] = {}
    for e in rows:
        totals[e["category"]] = totals.get(e["category"], 0) + e["amount_cents"]
    return "\n".join(f"{c}: {money(t)}" for c, t in sorted(totals.items(), key=lambda kv: -kv[1]))


@mcp.resource("expenses://{month}/summary")
def month_resource(month: str) -> str:
    """All expenses for a month as JSON."""
    import json
    return json.dumps([e for e in EXPENSES if e["date"].startswith(month)])


@mcp.prompt()
def budget_review(month: str) -> str:
    """Ask the assistant to review a month's spending."""
    return (f"Use monthly_summary and list_expenses for {month}. Identify the top 3 categories, "
            "flag anything unusual, and suggest one concrete saving.")


if __name__ == "__main__":
    mcp.run()                              # stdio by default
    # mcp.run(transport="streamable-http")  # remote; serves at http://localhost:8000/mcp
```

FastMCP derives the **JSON Schema from type hints + Pydantic** `Field`s and uses the **docstring as the tool description**. Type hints are the contract, so write them carefully. Sync and `async def` tools are both supported; use async for I/O.

## Run, inspect, install

```bash
mcp dev server.py                  # launches the MCP Inspector against the server
mcp install server.py              # register with Claude Desktop (stdio)
claude mcp add expenses -- python /abs/path/server.py
```
Streamable HTTP: `python server.py` after switching the transport, then Inspector → `http://localhost:8000/mcp`.

## Context, logging, progress

```python
from mcp.server.fastmcp import Context

@mcp.tool()
async def import_csv(path: str, ctx: Context) -> str:
    """Import expenses from a CSV file."""
    await ctx.info(f"Importing {path}")
    rows = load(path)
    for i, row in enumerate(rows):
        save(row)
        await ctx.report_progress(i + 1, len(rows))
    return f"Imported {len(rows)} rows"
```

## Reusing FastAPI code
Put business logic in a plain `app/services/expenses.py` (pure functions + repository), then expose it two ways: FastAPI routes for the React app and MCP tools for AI clients. **Same domain, two adapters**: a good architecture talking point (ports and adapters).

```python
from app.services.expenses import ExpenseService      # shared
service = ExpenseService(session_factory)

@mcp.tool()
async def list_expenses(month: str, ctx: Context) -> list[dict]:
    user_id = current_user_id(ctx)                    # derived from the authenticated request, not from args
    return [e.model_dump() for e in await service.list(user_id, month)]
```

You can also mount the MCP server inside a FastAPI app (`app.mount("/mcp", mcp.streamable_http_app())`) so one ECS service serves both the REST API and MCP.

## Testing

```python
import pytest
from mcp.shared.memory import create_connected_server_and_client_session

@pytest.mark.anyio
async def test_summary():
    async with create_connected_server_and_client_session(mcp._mcp_server) as client:
        res = await client.call_tool("monthly_summary", {"month": "2026-10"})
        assert not res.isError
        assert "transport" in res.content[0].text
```
(Helper names can shift between SDK versions; unit-testing the underlying functions directly is the most stable approach.)

## Exercise
Port `delete_expense` with a confirmation argument, add a `receipt_text` tool that calls Bedrock to extract fields from a receipt image/text (see [aws/07](../aws/07-bedrock-and-ai-integration.md)), and test with the Inspector.

## Interview Q&A
- **How does FastMCP generate the tool schema?** From function signature type hints/Pydantic models; docstring → description.
- **How do you keep REST and MCP in sync?** Shared service layer, two thin adapters.
- **How do you expose long-running work?** Progress notifications (`report_progress`), or return a job id + a `get_job_status` tool.
- **How do you attach the user identity in a remote server?** From validated bearer token on the HTTP request (see [05](05-security-and-deployment-on-aws.md)).
