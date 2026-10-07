# FastAPI 07 — Refactoring to Clean Architecture and SOLID

Theory and rules: [../fullstack/08](../fullstack/08-solid-and-clean-architecture.md). This tutorial is the **reference structure for `api-fastapi/`**. Tutorials 01–05 introduced the pieces (routers, Pydantic, SQLAlchemy, auth, deployment); here they are placed in the right layers. If you are building from scratch, use this layout from the start and treat 01–05 as the "how" for the adapter code.

## Target layout

```
api-fastapi/
  app/
    domain/                      # pure Python. No fastapi, pydantic, boto3.
      errors.py  money.py  expense.py  summary.py
    application/                 # use cases + ports. Imports domain only.
      ports.py                   # Protocols: ExpenseReader/Writer, Clock, IdGenerator, InsightGenerator, ReceiptStorage
      dto.py                     # use-case inputs/outputs (plain dataclasses)
      create_expense.py  get_expense.py  list_expenses.py  delete_expense.py
      monthly_summary.py  generate_insight.py  create_upload_url.py
      use_cases.py               # UseCases: builds every use case from ports (knows no adapter)
    adapters/                    # implement ports. Import application + domain + their SDK.
      persistence/dynamodb_expense_repo.py   memory_expense_repo.py
      system.py                              # SystemClock, UlidGenerator
      llm/bedrock_insight_generator.py
      storage/s3_receipt_storage.py
    interface/http/              # driving adapter (FastAPI). Imports application + domain.
      schemas.py  routers.py  deps.py  errors.py
    container.py                 # composition root: the ONLY module that names concrete adapters
    main.py                      # create_app(use_cases, dev_user_id)
    dev.py                       # no-AWS entry point: uvicorn app.dev:app
  tests/
    domain/  application/  adapters/ (contract)  interface/
```
Dependency direction: `interface → application → domain` and `adapters → application → domain`; `container` and `main` see everything.

> **This layout is implemented and tested in [`api-fastapi/`](../../api-fastapi).** The code blocks below are copied from those files. One design lesson from building it: the HTTP layer must not import the composition root, so the object that *builds use cases from ports* (`UseCases`) lives in `application/`, and only `container.py` (which imports concrete adapters) is outermost. `import-linter` caught the first version of this tutorial getting that wrong.

## Domain (pure)

```python
# app/domain/errors.py
class DomainError(Exception):
    code = "domain_error"
    def __init__(self, message: str):
        super().__init__(message)
        self.message = message

class InvalidExpense(DomainError):
    code = "invalid_expense"

class ExpenseNotFound(DomainError):
    code = "not_found"
```

```python
# app/domain/money.py
from dataclasses import dataclass
from app.domain.errors import InvalidExpense

SUPPORTED = {"USD", "EUR", "MXN"}

@dataclass(frozen=True, slots=True)
class Money:
    """Value object: immutable, validated on construction, compared by value."""
    cents: int
    currency: str = "USD"

    def __post_init__(self) -> None:
        if isinstance(self.cents, bool) or not isinstance(self.cents, int) or self.cents <= 0:
            raise InvalidExpense("amount must be a positive integer number of cents")
        if self.currency not in SUPPORTED:
            raise InvalidExpense(f"unsupported currency {self.currency!r}")

    def __add__(self, other: "Money") -> "Money":
        if other.currency != self.currency:
            raise InvalidExpense("cannot add different currencies")
        return Money(self.cents + other.cents, self.currency)
```

```python
# app/domain/expense.py
from dataclasses import dataclass
from datetime import date, datetime
from enum import StrEnum
from app.domain.errors import InvalidExpense
from app.domain.money import Money

class Category(StrEnum):
    FOOD = "food"; TRANSPORT = "transport"; HOUSING = "housing"; FUN = "fun"; OTHER = "other"

    @classmethod
    def parse(cls, raw: str) -> "Category":
        try:
            return cls(raw)
        except ValueError:
            raise InvalidExpense(f"unknown category {raw!r}") from None

@dataclass(frozen=True, slots=True)
class Expense:
    id: str
    user_id: str
    amount: Money
    category: Category
    spent_on: date
    created_at: datetime
    description: str | None = None

    @classmethod
    def create(cls, *, id: str, user_id: str, amount: Money, category: Category, spent_on: date,
               today: date, now: datetime, description: str | None = None) -> "Expense":
        """Factory enforcing invariants. `today`/`now` are passed in (no hidden clock → deterministic)."""
        if spent_on > today:
            raise InvalidExpense("date cannot be in the future")
        desc = (description or "").strip() or None
        if desc and len(desc) > 200:
            raise InvalidExpense("description too long (max 200)")
        return cls(id=id, user_id=user_id, amount=amount, category=category,
                   spent_on=spent_on, created_at=now, description=desc)
```

```python
# app/domain/summary.py   (domain service: pure function)
from collections import defaultdict
from collections.abc import Iterable
from app.domain.expense import Expense

def summarize(expenses: Iterable[Expense]) -> dict[str, int]:
    """Total cents per category, largest first."""
    totals: dict[str, int] = defaultdict(int)
    for e in expenses:
        totals[e.category.value] += e.amount.cents
    return dict(sorted(totals.items(), key=lambda kv: -kv[1]))
```
**SRP**: each file has one reason to change. **No I/O, no framework**: unit tests need nothing.

## Application: ports (ISP) and use cases (DIP)

```python
# app/application/ports.py
from datetime import date, datetime
from typing import Protocol
from app.application.dto import Insight
from app.domain.expense import Expense

class ExpenseReader(Protocol):
    def get(self, user_id: str, expense_id: str) -> Expense | None: ...
    def list_range(self, user_id: str, start: date, end: date, limit: int,
                   cursor: str | None) -> tuple[list[Expense], str | None]: ...      # cursor is OPAQUE

class ExpenseWriter(Protocol):
    def add(self, expense: Expense) -> None: ...
    def delete(self, user_id: str, expense_id: str) -> None: ...

class Clock(Protocol):
    def today(self) -> date: ...
    def now(self) -> datetime: ...

class IdGenerator(Protocol):
    def new_id(self) -> str: ...

class InsightGenerator(Protocol):
    def generate(self, month: str, totals: dict[str, int]) -> Insight: ...

class ReceiptStorage(Protocol):
    def upload_url(self, user_id: str, content_type: str) -> tuple[str, str]: ...   # (url, key)
```
Segregated ports (ISP): `CreateExpense` needs only `ExpenseWriter`, `Clock`, `IdGenerator`. `ListExpenses` needs only `ExpenseReader`. One concrete repo class implements both.

```python
# app/application/dto.py
from dataclasses import dataclass, field
from datetime import date
from app.domain.expense import Expense

@dataclass(frozen=True)
class CreateExpenseInput:
    user_id: str
    amount_cents: int
    currency: str
    category: str
    spent_on: date
    description: str | None = None

@dataclass(frozen=True)
class ListExpensesInput:
    user_id: str
    start: date
    end: date
    limit: int = 50
    cursor: str | None = None

@dataclass(frozen=True)
class Page:
    items: list[Expense]
    next_cursor: str | None

@dataclass(frozen=True)
class Insight:
    summary: str
    anomalies: list[str] = field(default_factory=list)
    tips: list[str] = field(default_factory=list)
```

```python
# app/application/create_expense.py
from app.application.dto import CreateExpenseInput
from app.application.ports import Clock, ExpenseWriter, IdGenerator
from app.domain.expense import Category, Expense
from app.domain.money import Money

class CreateExpense:
    def __init__(self, writer: ExpenseWriter, ids: IdGenerator, clock: Clock) -> None:
        self._writer, self._ids, self._clock = writer, ids, clock

    def __call__(self, inp: CreateExpenseInput) -> Expense:
        expense = Expense.create(
            id=self._ids.new_id(), user_id=inp.user_id,
            amount=Money(inp.amount_cents, inp.currency),
            category=Category.parse(inp.category),
            spent_on=inp.spent_on, today=self._clock.today(), now=self._clock.now(),
            description=inp.description)
        self._writer.add(expense)
        return expense
```

```python
# app/application/get_expense.py
from app.application.ports import ExpenseReader
from app.domain.errors import ExpenseNotFound
from app.domain.expense import Expense

class GetExpense:
    def __init__(self, reader: ExpenseReader) -> None:
        self._reader = reader

    def __call__(self, user_id: str, expense_id: str) -> Expense:
        expense = self._reader.get(user_id, expense_id)
        if expense is None:                       # same answer for "missing" and "someone else's" (no existence leak)
            raise ExpenseNotFound("expense not found")
        return expense
```

```python
# app/application/list_expenses.py
from app.application.dto import ListExpensesInput, Page
from app.application.ports import ExpenseReader
from app.domain.errors import InvalidExpense

class ListExpenses:
    def __init__(self, reader: ExpenseReader) -> None:
        self._reader = reader

    def __call__(self, inp: ListExpensesInput) -> Page:
        if inp.end < inp.start:
            raise InvalidExpense("'to' must not be before 'from'")
        limit = max(1, min(inp.limit, 100))            # application rule: cap page size
        items, nxt = self._reader.list_range(inp.user_id, inp.start, inp.end, limit, inp.cursor)
        return Page(items, nxt)
```

```python
# app/application/monthly_summary.py
import calendar
from datetime import date
from app.application.ports import ExpenseReader
from app.domain.summary import summarize

class MonthlySummary:
    def __init__(self, reader: ExpenseReader) -> None:
        self._reader = reader

    def __call__(self, user_id: str, month: str) -> dict[str, int]:
        year, mon = map(int, month.split("-"))
        start, end = date(year, mon, 1), date(year, mon, calendar.monthrange(year, mon)[1])
        items, cursor = [], None
        while True:                                      # drain pages
            page, cursor = self._reader.list_range(user_id, start, end, 100, cursor)
            items += page
            if cursor is None:
                return summarize(items)
```

```python
# app/application/generate_insight.py   (OCP: provider is a port)
from app.application.dto import Insight
from app.application.monthly_summary import MonthlySummary
from app.application.ports import InsightGenerator
from app.domain.errors import ExpenseNotFound

class GenerateInsight:
    def __init__(self, summary: MonthlySummary, generator: InsightGenerator) -> None:
        self._summary, self._generator = summary, generator

    def __call__(self, user_id: str, month: str) -> Insight:
        totals = self._summary(user_id, month)
        if not totals:
            raise ExpenseNotFound(f"no expenses in {month}")
        return self._generator.generate(month, totals)       # Bedrock, Azure, or a fake: this line never changes
```
Use cases return **domain objects/DTOs**, raise **domain errors**, and know nothing about HTTP.

## Adapters (driven)

```python
# app/adapters/persistence/dynamodb_expense_repo.py
import base64, json, os
from datetime import date, datetime
from boto3.dynamodb.conditions import Key
import boto3
from app.domain.expense import Category, Expense
from app.domain.money import Money

class DynamoDbExpenseRepo:                      # implements ExpenseReader + ExpenseWriter (structural typing)
    def __init__(self, table) -> None:
        self._t = table

    @classmethod
    def from_env(cls) -> "DynamoDbExpenseRepo":
        ddb = boto3.resource("dynamodb", endpoint_url=os.getenv("DYNAMODB_ENDPOINT"))
        return cls(ddb.Table(os.environ["TABLE_NAME"]))

    # ---- mapping: domain <-> storage item (the only code that knows key design) ----
    @staticmethod
    def _to_item(e: Expense) -> dict:
        return {"PK": f"USER#{e.user_id}", "SK": f"EXP#{e.id}",
                "GSI1PK": f"USER#{e.user_id}", "GSI1SK": f"DATE#{e.spent_on.isoformat()}#{e.id}",
                "id": e.id, "userId": e.user_id, "amountCents": e.amount.cents, "currency": e.amount.currency,
                "category": e.category.value, "spentOn": e.spent_on.isoformat(),
                "createdAt": e.created_at.isoformat(), **({"description": e.description} if e.description else {})}

    @staticmethod
    def _from_item(i: dict) -> Expense:
        return Expense(id=i["id"], user_id=i["userId"], amount=Money(int(i["amountCents"]), i["currency"]),
                       category=Category(i["category"]), spent_on=date.fromisoformat(i["spentOn"]),
                       created_at=datetime.fromisoformat(i["createdAt"]), description=i.get("description"))

    def add(self, expense: Expense) -> None:
        self._t.put_item(Item=self._to_item(expense), ConditionExpression="attribute_not_exists(PK)")

    def get(self, user_id: str, expense_id: str) -> Expense | None:
        r = self._t.get_item(Key={"PK": f"USER#{user_id}", "SK": f"EXP#{expense_id}"})
        return self._from_item(r["Item"]) if "Item" in r else None

    def delete(self, user_id: str, expense_id: str) -> None:
        self._t.delete_item(Key={"PK": f"USER#{user_id}", "SK": f"EXP#{expense_id}"})

    def list_range(self, user_id, start, end, limit, cursor):
        kw = dict(IndexName="GSI1", ScanIndexForward=False, Limit=limit,
                  KeyConditionExpression=Key("GSI1PK").eq(f"USER#{user_id}")
                  & Key("GSI1SK").between(f"DATE#{start.isoformat()}", f"DATE#{end.isoformat()}~"))
        if cursor:
            kw["ExclusiveStartKey"] = json.loads(base64.urlsafe_b64decode(cursor))
        r = self._t.query(**kw)
        nxt = base64.urlsafe_b64encode(json.dumps(r["LastEvaluatedKey"]).encode()).decode() if "LastEvaluatedKey" in r else None
        return [self._from_item(i) for i in r["Items"]], nxt          # opaque cursor: no DynamoDB types leak
```

```python
# app/adapters/persistence/memory_expense_repo.py
import base64
from app.domain.expense import Expense

class InMemoryExpenseRepo:
    def __init__(self) -> None:
        self._by_key: dict[tuple[str, str], Expense] = {}

    def add(self, e: Expense) -> None:
        self._by_key[(e.user_id, e.id)] = e

    def get(self, user_id, expense_id):
        return self._by_key.get((user_id, expense_id))

    def delete(self, user_id, expense_id) -> None:
        self._by_key.pop((user_id, expense_id), None)

    def list_range(self, user_id, start, end, limit, cursor):
        rows = sorted((e for (u, _), e in self._by_key.items() if u == user_id and start <= e.spent_on <= end),
                      key=lambda e: (e.spent_on, e.id), reverse=True)
        offset = int(base64.urlsafe_b64decode(cursor)) if cursor else 0
        page = rows[offset: offset + limit]
        nxt = base64.urlsafe_b64encode(str(offset + limit).encode()).decode() if offset + limit < len(rows) else None
        return page, nxt
```

```python
# app/adapters/system.py
from datetime import date, datetime, timezone
from ulid import ULID

class SystemClock:
    def today(self) -> date: return datetime.now(timezone.utc).date()
    def now(self) -> datetime: return datetime.now(timezone.utc)

class UlidGenerator:
    def new_id(self) -> str: return str(ULID())
```

```python
# app/adapters/llm/bedrock_insight_generator.py   (OCP: add azure_insight_generator.py beside it)
import json
import boto3
from botocore.config import Config
from app.application.dto import Insight

class BedrockInsightGenerator:
    def __init__(self, model_id: str, client=None) -> None:
        self._model_id = model_id
        self._client = client or boto3.client("bedrock-runtime", config=Config(read_timeout=60, retries={"max_attempts": 2}))

    def generate(self, month: str, totals: dict[str, int]) -> Insight:
        prompt = (f"Month: {month}\n<totals>{json.dumps(totals)}</totals>\n"
                  'Return ONLY JSON: {"summary": str, "anomalies": [str], "tips": [str]}. Amounts are cents.')
        r = self._client.converse(modelId=self._model_id,
                                  system=[{"text": "You analyze personal spending using only the provided data."}],
                                  messages=[{"role": "user", "content": [{"text": prompt}]}],
                                  inferenceConfig={"maxTokens": 600, "temperature": 0.2})
        data = json.loads(r["output"]["message"]["content"][0]["text"])
        return Insight(summary=str(data["summary"]), anomalies=list(data.get("anomalies", [])), tips=list(data.get("tips", []))[:5])
```
Everything AWS-specific is quarantined in `adapters/`. Mapping domain↔item lives in one place.

## Interface adapter: HTTP (FastAPI)

```python
# app/interface/http/schemas.py   (Pydantic lives HERE, not in domain)
from datetime import date, datetime

from pydantic import BaseModel, ConfigDict, Field
from pydantic.alias_generators import to_camel

from app.application.dto import Insight
from app.domain.expense import Expense


class Camel(BaseModel):
    model_config = ConfigDict(alias_generator=to_camel, populate_by_name=True)


class CreateExpenseRequest(Camel):
    amount_cents: int = Field(gt=0)
    currency: str = "USD"
    category: str
    description: str | None = Field(default=None, max_length=200)
    spent_on: date = Field(alias="date")


class ExpenseResponse(Camel):
    id: str
    amount_cents: int
    currency: str
    category: str
    description: str | None = None
    spent_on: date = Field(alias="date")
    created_at: datetime

    @classmethod
    def from_domain(cls, e: Expense) -> "ExpenseResponse":
        return cls(
            id=e.id, amount_cents=e.amount.cents, currency=e.amount.currency, category=e.category.value,
            description=e.description, spent_on=e.spent_on, created_at=e.created_at,
        )


class PageResponse(Camel):
    items: list[ExpenseResponse]
    next_cursor: str | None = None


class SummaryResponse(Camel):
    month: str
    totals: dict[str, int]


class InsightResponse(Camel):
    summary: str
    anomalies: list[str]
    tips: list[str]

    @classmethod
    def from_domain(cls, i: Insight) -> "InsightResponse":
        return cls(summary=i.summary, anomalies=i.anomalies, tips=i.tips)


class UploadUrlRequest(Camel):
    content_type: str


class UploadUrlResponse(Camel):
    url: str
    key: str
```

```python
# app/interface/http/deps.py   (identity comes from the gateway-verified claim; the use cases only see a plain user_id)
import json

from fastapi import Depends, Header, HTTPException, Request

from app.application.use_cases import UseCases


def use_cases(request: Request) -> UseCases:
    return request.app.state.use_cases


def current_user_id(
    request: Request,
    x_amzn_request_context: str | None = Header(default=None),
) -> str:
    """Identity from the API Gateway request context forwarded by the Lambda Web Adapter.

    The gateway already validated the JWT; this app only reads the verified claim.
    """
    if x_amzn_request_context:
        try:
            return json.loads(x_amzn_request_context)["authorizer"]["jwt"]["claims"]["sub"]
        except (KeyError, TypeError, ValueError):
            raise HTTPException(401, "unauthenticated") from None
    if request.app.state.dev_user_id:  # local development only
        return request.app.state.dev_user_id
    raise HTTPException(401, "unauthenticated")


__all__ = ["Depends", "use_cases", "current_user_id"]
```

```python
# app/interface/http/routers.py   (each handler: translate in, call ONE use case, translate out)
from datetime import date
from typing import Annotated

from fastapi import APIRouter, Depends, Query, Response, status

from app.application.dto import CreateExpenseInput, ListExpensesInput
from app.application.monthly_summary import month_bounds
from app.application.use_cases import UseCases
from app.interface.http.deps import current_user_id, use_cases
from app.interface.http.schemas import (
    CreateExpenseRequest,
    ExpenseResponse,
    InsightResponse,
    PageResponse,
    SummaryResponse,
    UploadUrlRequest,
    UploadUrlResponse,
)

expenses = APIRouter(prefix="/expenses", tags=["expenses"])
reports = APIRouter(prefix="/reports", tags=["reports"])
receipts = APIRouter(prefix="/receipts", tags=["receipts"])


@expenses.post("", response_model=ExpenseResponse, response_model_by_alias=True, status_code=status.HTTP_201_CREATED)
def create_expense(
    body: CreateExpenseRequest,
    response: Response,
    user_id: str = Depends(current_user_id),
    c: UseCases = Depends(use_cases),
):
    expense = c.create_expense(
        CreateExpenseInput(
            user_id=user_id, amount_cents=body.amount_cents, currency=body.currency,
            category=body.category, spent_on=body.spent_on, description=body.description,
        )
    )
    response.headers["Location"] = f"/expenses/{expense.id}"
    return ExpenseResponse.from_domain(expense)


@expenses.get("", response_model=PageResponse, response_model_by_alias=True)
def list_expenses(
    start: Annotated[date, Query(alias="from")],
    end: Annotated[date, Query(alias="to")],
    limit: int = 50,
    cursor: str | None = None,
    category: str | None = None,
    user_id: str = Depends(current_user_id),
    c: UseCases = Depends(use_cases),
):
    page = c.list_expenses(ListExpensesInput(user_id=user_id, start=start, end=end, limit=limit, cursor=cursor))
    items = [e for e in page.items if category is None or e.category.value == category]  # filter after read
    return PageResponse(items=[ExpenseResponse.from_domain(e) for e in items], next_cursor=page.next_cursor)


@expenses.get("/{expense_id}", response_model=ExpenseResponse, response_model_by_alias=True)
def get_expense(expense_id: str, user_id: str = Depends(current_user_id), c: UseCases = Depends(use_cases)):
    return ExpenseResponse.from_domain(c.get_expense(user_id, expense_id))


@expenses.delete("/{expense_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_expense(expense_id: str, user_id: str = Depends(current_user_id), c: UseCases = Depends(use_cases)):
    c.delete_expense(user_id, expense_id)
    return Response(status_code=status.HTTP_204_NO_CONTENT)


@reports.get("/summary", response_model=SummaryResponse, response_model_by_alias=True)
def monthly_summary(month: str, user_id: str = Depends(current_user_id), c: UseCases = Depends(use_cases)):
    month_bounds(month)  # validates shape → InvalidExpense → 422
    return SummaryResponse(month=month, totals=c.monthly_summary(user_id, month))


@reports.post("/insights", response_model=InsightResponse, response_model_by_alias=True)
def insights(month: str, user_id: str = Depends(current_user_id), c: UseCases = Depends(use_cases)):
    return InsightResponse.from_domain(c.generate_insight(user_id, month))


@receipts.post("/upload-url", response_model=UploadUrlResponse, response_model_by_alias=True)
def upload_url(body: UploadUrlRequest, user_id: str = Depends(current_user_id), c: UseCases = Depends(use_cases)):
    url, key = c.create_upload_url(user_id, body.content_type)
    return UploadUrlResponse(url=url, key=key)
```

```python
# app/interface/http/errors.py   (domain error → HTTP: the ONLY place status codes meet domain errors)
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

from app.domain.errors import DomainError, ExpenseNotFound


def install_error_handlers(app: FastAPI) -> None:
    @app.exception_handler(DomainError)
    async def _domain(_: Request, exc: DomainError):
        status = 404 if isinstance(exc, ExpenseNotFound) else 422
        return JSONResponse(
            status_code=status,
            media_type="application/problem+json",
            content={
                "type": f"https://api.example.com/problems/{exc.code}",
                "title": exc.message,
                "status": status,
            },
        )
```

## Composition

```python
# app/application/use_cases.py   (builds use cases from ports; imports no adapter, so controllers may depend on it)
from functools import cached_property

from app.application.create_expense import CreateExpense
from app.application.create_upload_url import CreateUploadUrl
from app.application.delete_expense import DeleteExpense
from app.application.generate_insight import GenerateInsight
from app.application.get_expense import GetExpense
from app.application.list_expenses import ListExpenses
from app.application.monthly_summary import MonthlySummary


class UseCases:
    """Builds every use case from ports. Knows no concrete adapter, so controllers may depend on it.

    The composition root (app/container.py) decides which adapters are passed in.
    """

    def __init__(self, *, repo, ids, clock, insights, receipts) -> None:
        self.repo, self.ids, self.clock = repo, ids, clock
        self.insights, self.receipts = insights, receipts

    @cached_property
    def create_expense(self) -> CreateExpense:
        return CreateExpense(self.repo, self.ids, self.clock)

    @cached_property
    def get_expense(self) -> GetExpense:
        return GetExpense(self.repo)

    @cached_property
    def list_expenses(self) -> ListExpenses:
        return ListExpenses(self.repo)

    @cached_property
    def delete_expense(self) -> DeleteExpense:
        return DeleteExpense(self.repo, self.repo)

    @cached_property
    def monthly_summary(self) -> MonthlySummary:
        return MonthlySummary(self.repo)

    @cached_property
    def generate_insight(self) -> GenerateInsight:
        return GenerateInsight(self.monthly_summary, self.insights)

    @cached_property
    def create_upload_url(self) -> CreateUploadUrl:
        return CreateUploadUrl(self.receipts)
```

```python
# app/container.py   (composition root: the ONLY module that names concrete adapters)
"""Composition root: the ONLY module that names concrete adapters.

Tests build `UseCases` with fakes; production builds it here.
"""
import os

from app.application.use_cases import UseCases


def build_production_use_cases() -> UseCases:
    from app.adapters.llm.bedrock_insight_generator import BedrockInsightGenerator
    from app.adapters.persistence.dynamodb_expense_repo import DynamoDbExpenseRepo
    from app.adapters.storage.s3_receipt_storage import S3ReceiptStorage
    from app.adapters.system import SystemClock, UlidGenerator

    return UseCases(
        repo=DynamoDbExpenseRepo.from_env(),
        ids=UlidGenerator(),
        clock=SystemClock(),
        insights=BedrockInsightGenerator(os.environ["BEDROCK_MODEL_ID"]) if os.getenv("BEDROCK_MODEL_ID") else None,
        receipts=S3ReceiptStorage.from_env() if os.getenv("RECEIPTS_BUCKET") else None,
    )
```

```python
# app/main.py
import os

from fastapi import FastAPI

from app.application.use_cases import UseCases
from app.interface.http.errors import install_error_handlers
from app.interface.http.routers import expenses, receipts, reports


def create_app(use_cases: UseCases | None = None, dev_user_id: str | None = None) -> FastAPI:
    if use_cases is None:  # production wiring; imported lazily so tests never touch AWS adapters
        from app.container import build_production_use_cases

        use_cases = build_production_use_cases()
        dev_user_id = dev_user_id or os.getenv("DEV_USER_ID")

    app = FastAPI(title="Expense Tracker API", version="1.0.0")
    app.state.use_cases = use_cases
    app.state.dev_user_id = dev_user_id
    install_error_handlers(app)
    app.include_router(expenses)
    app.include_router(reports)
    app.include_router(receipts)

    @app.get("/health", tags=["ops"])
    def health() -> dict[str, str]:
        return {"status": "ok"}

    return app
```

Run it with an app **factory** so importing the module has no side effects (no AWS clients at import time):
`uvicorn app.main:create_app --factory --host 0.0.0.0 --port 8080` (the Dockerfile `CMD`). For work without AWS,
`app/dev.py` builds the same app around in-memory fakes: `DEV_USER_ID=u1 uvicorn app.dev:app --port 8080`.

## Tests, per layer

**Domain** (no fixtures at all):
```python
def test_future_date_rejected():
    with pytest.raises(InvalidExpense):
        Expense.create(id="1", user_id="u", amount=Money(100), category=Category.FOOD,
                       spent_on=date(2026, 10, 7), today=date(2026, 10, 6), now=datetime(2026, 10, 6))

def test_summarize_orders_by_total():
    assert list(summarize([exp("food", 500), exp("fun", 900)])) == ["fun", "food"]
```
**Use case** (fakes, milliseconds):
```python
class FixedClock:
    def today(self): return date(2026, 10, 6)
    def now(self): return datetime(2026, 10, 6, 12, tzinfo=timezone.utc)
class SeqIds:
    def __init__(self): self.n = 0
    def new_id(self): self.n += 1; return f"e{self.n}"

def test_create_persists_and_returns():
    repo = InMemoryExpenseRepo()
    uc = CreateExpense(repo, SeqIds(), FixedClock())
    e = uc(CreateExpenseInput("u1", 1250, "USD", "food", date(2026, 10, 5), " lunch "))
    assert e.description == "lunch" and repo.get("u1", "e1") == e

def test_cannot_read_other_users_expense():
    repo = InMemoryExpenseRepo(); CreateExpense(repo, SeqIds(), FixedClock())(CreateExpenseInput("alice", 100, "USD", "food", date(2026, 10, 1)))
    with pytest.raises(ExpenseNotFound):
        GetExpense(repo)("bob", "e1")
```
**LSP contract test**, one suite, every implementation:
```python
# tests/adapters/test_expense_repo_contract.py
@pytest.fixture(params=["memory", "dynamodb"])
def repo(request, ddb_table):                       # ddb_table fixture talks to DynamoDB Local; skip if unavailable
    return InMemoryExpenseRepo() if request.param == "memory" else DynamoDbExpenseRepo(ddb_table)

def test_roundtrip(repo):
    e = make_expense("u1", "e1"); repo.add(e)
    assert repo.get("u1", "e1") == e

def test_other_user_cannot_see(repo):
    repo.add(make_expense("alice", "e1"))
    assert repo.get("bob", "e1") is None
    assert repo.list_range("bob", date(2000, 1, 1), date(2100, 1, 1), 10, None)[0] == []

def test_list_newest_first_and_pages_without_gaps(repo):
    for i in range(5): repo.add(make_expense("u1", f"e{i}", spent_on=date(2026, 10, 1 + i)))
    p1, c = repo.list_range("u1", date(2026, 10, 1), date(2026, 10, 31), 3, None)
    p2, c2 = repo.list_range("u1", date(2026, 10, 1), date(2026, 10, 31), 3, c)
    assert [e.id for e in p1 + p2] == ["e4", "e3", "e2", "e1", "e0"] and c2 is None
```
**HTTP adapter** (build the real app around fakes; no AWS):
```python
@pytest.fixture
def client():
    uc = UseCases(repo=InMemoryExpenseRepo(), ids=SeqIds(), clock=FixedClock(), insights=FakeInsights(), receipts=FakeReceipts())
    return TestClient(create_app(uc, dev_user_id="u1"))

def test_validation_error_maps_to_422(client):
    r = client.post("/expenses", json={"amountCents": 100, "category": "bogus", "date": "2026-10-01"})
    assert r.status_code == 422 and r.json()["type"].endswith("invalid_expense")
```
No `dependency_overrides` gymnastics: `create_app(use_cases)` *is* the override point.

## Enforce the layers (CI)

```toml
# pyproject.toml  (pip install import-linter)  -- the real file is api-fastapi/pyproject.toml
[tool.importlinter]
root_package = "app"
include_external_packages = true     # required for the "forbidden" contract on third-party packages

[[tool.importlinter.contracts]]
name = "Clean architecture layers"
type = "layers"
layers = ["app.container", "app.interface", "app.adapters", "app.application", "app.domain"]
exhaustive = false

[[tool.importlinter.contracts]]
name = "Interface and adapters are independent"
type = "independence"
modules = ["app.interface", "app.adapters"]

[[tool.importlinter.contracts]]
name = "Domain and application are framework-free"
type = "forbidden"
source_modules = ["app.domain", "app.application"]
forbidden_modules = ["fastapi", "pydantic", "boto3", "botocore", "starlette", "ulid"]
```
Run `lint-imports` in the pipeline ([../infra/07](../infra/07-cicd-github-actions.md)).

## Mapping of the earlier tutorials
| Earlier tutorial content | Where it lives now |
|---|---|
| 01 routers, `Depends`, middleware, settings | `interface/http`, `container.py`, `main.py` |
| 02 Pydantic models | `interface/http/schemas.py` (transport only); invariants moved to `domain` |
| 03 SQLAlchemy repository | an alternative adapter `adapters/persistence/sqlalchemy_expense_repo.py` implementing the same ports (a live demonstration of OCP/LSP: run the contract tests on it too) |
| 04 JWT verification / identity | `interface/http/deps.py` (the user id enters the use case as plain input) |
| 05 Bedrock endpoint | `adapters/llm/` + `GenerateInsight` use case |

## SOLID scorecard for this codebase
| Principle | Where you can point |
|---|---|
| SRP | controller / use case / entity / repo each change for one reason |
| OCP | new LLM provider or DB = new adapter + one line in `container.py` |
| LSP | one contract suite passes for memory and DynamoDB repos |
| ISP | `CreateExpense` depends on `ExpenseWriter`, `Clock`, `IdGenerator` only |
| DIP | use cases import `ports.py`, never `boto3`; `container.py` wires |

## Exercise
1. Implement the missing use cases (`DeleteExpense`, `CreateUploadUrl`) and their tests with fakes.
2. Add `SqlAlchemyExpenseRepo` and run the contract suite against it.
3. Add `AzureFoundryInsightGenerator` (stub) and select the provider with an env var in `build_production_use_cases()`, touching nothing else.
4. Add the `import-linter` contracts and deliberately import `boto3` in `domain/` to watch CI fail.

## Interview Q&A
- **Where does validation live?** Shape in Pydantic schemas; invariants in `Money`/`Expense`; ownership in the use case + repo scoping.
- **Why not use Pydantic models as domain entities?** They tie the domain to a library and a serialization concern; domain stays dependency-free and constructed only through its factory.
- **How do you test without AWS?** Fake ports in use-case tests; contract suite hits DynamoDB Local only for the real adapter.
- **What's the cost?** More files and mappers; justified by swappability (DynamoDB/Postgres, Bedrock/Azure), testability and multiple inbound adapters (REST + MCP).
- **`Protocol` vs `ABC`?** `Protocol` gives structural typing: adapters don't import the application layer's base class, so the dependency stays one-directional.
