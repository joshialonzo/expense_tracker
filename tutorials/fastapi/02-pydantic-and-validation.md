# FastAPI 02 — Pydantic v2 Models, Validation and Serialization

## Why it matters
Validation at the boundary is where APIs get safe. Pydantic is also the Python analogue of Zod ([../typescript/01](../typescript/01-types-fundamentals.md)).

> **Clean-architecture note:** Pydantic models are *transport* schemas and belong in `interface/http/schemas.py`. Business invariants (positive money, no future dates) live in `domain/` ([07](07-clean-architecture.md)); the schema validates shape, the domain validates meaning.

## Schemas for the tracker

```python
# app/schemas.py
from datetime import date, datetime
from typing import Annotated, Literal
from uuid import UUID
from pydantic import BaseModel, ConfigDict, Field, field_validator, model_validator
from pydantic.alias_generators import to_camel

Currency = Literal["USD", "EUR", "MXN"]
Category = Literal["food", "transport", "housing", "fun", "other"]

class CamelModel(BaseModel):
    """JSON in camelCase (JS-friendly), snake_case in Python."""
    model_config = ConfigDict(alias_generator=to_camel, populate_by_name=True, from_attributes=True)

class ExpenseBase(CamelModel):
    amount_cents: Annotated[int, Field(gt=0, le=10_000_000_00, description="Amount in minor units")]
    currency: Currency = "USD"
    category: Category
    description: Annotated[str | None, Field(max_length=200)] = None
    spent_on: Annotated[date, Field(alias="date")]      # wire name "date", matches ORM column spent_on

    @field_validator("description")
    @classmethod
    def strip_description(cls, v: str | None) -> str | None:
        return v.strip() or None if v else v

    @model_validator(mode="after")
    def not_in_future(self):
        if self.spent_on > date.today():
            raise ValueError("date cannot be in the future")
        return self

class ExpenseCreate(ExpenseBase):
    pass

class ExpenseUpdate(CamelModel):                 # PATCH: every field optional
    amount_cents: Annotated[int, Field(gt=0)] | None = None
    category: Category | None = None
    description: Annotated[str | None, Field(max_length=200)] = None
    spent_on: Annotated[date | None, Field(alias="date")] = None

class ExpenseOut(ExpenseBase):
    id: UUID
    receipt_key: str | None = None
    created_at: datetime

class ExpensePage(CamelModel):
    items: list[ExpenseOut]
    next_cursor: str | None = None
```

Key points:
- **Separate input/output models**: `ExpenseCreate` can't contain `id`, `userId` or `createdAt`, so **mass assignment is impossible**. `response_model=ExpenseOut` also **filters** output (never leaks DB columns like `user_id` or password hashes).
- `from_attributes=True` lets Pydantic read ORM objects (was `orm_mode`).
- Avoid naming a field `date` with type `date` (the name shadows the type inside the class body); use `spent_on` with `alias="date"`.
- `alias_generator=to_camel`: wire format `amountCents`, Python `amount_cents`. Return with `response_model_by_alias=True` (the default).
- Partial updates: `body.model_dump(exclude_unset=True)` gives only the fields the client sent (distinguishes "not provided" vs `null`).

```python
@router.patch("/{expense_id}", response_model=ExpenseOut)
def update_expense(expense_id: UUID, body: ExpenseUpdate, ...):
    changes = body.model_dump(exclude_unset=True)
    ...
```

## Validation errors
Invalid input → automatic `422` with details (`loc`, `msg`, `type`). Customize to a Problem Details shape:

```python
from fastapi.exceptions import RequestValidationError

@app.exception_handler(RequestValidationError)
async def validation_handler(request, exc: RequestValidationError):
    return JSONResponse(status_code=422, media_type="application/problem+json", content={
        "type": "https://api.example.com/problems/validation-error", "title": "Validation failed", "status": 422,
        "errors": [{"field": ".".join(str(p) for p in e["loc"][1:]), "message": e["msg"]} for e in exc.errors()],
    })
```

## Money correctly
Integer cents in the model (`int`) → exact. If you must accept decimals use `Decimal`, never `float`, and convert at the boundary. Format on the client with `Intl.NumberFormat`.

## Other useful features
- `Field(min_length=…, pattern=…)`, `EmailStr` (needs `email-validator`), `HttpUrl`, `UUID`, `constr`.
- `computed_field`, `model_validator(mode="before")` to normalize input.
- Discriminated unions: `Annotated[Card | Bank, Field(discriminator="kind")]` (mirrors TS discriminated unions).
- Generics: `class Page(BaseModel, Generic[T])` for reusable pagination envelopes.
- `TypeAdapter(list[ExpenseOut]).validate_python(data)` to validate arbitrary data (e.g. LLM JSON from Bedrock).

### Validating LLM output with Pydantic (ties to AWS 07)

```python
class Insight(BaseModel):
    summary: str
    anomalies: list[str] = []
    tips: Annotated[list[str], Field(max_length=5)] = []

def parse_insight(raw_text: str) -> Insight:
    try:
        return Insight.model_validate_json(raw_text)
    except ValidationError:
        # retry once with the validation error appended to the prompt, else fall back
        raise
```

## Testing the validation

```python
def test_rejects_negative_amount(client, auth_headers):
    r = client.post("/expenses", json={"amountCents": -5, "category": "food", "date": "2026-10-01"}, headers=auth_headers)
    assert r.status_code == 422
    assert r.json()["errors"][0]["field"] == "amountCents"

def test_unknown_field_is_ignored_or_rejected(client, auth_headers):
    # set model_config extra="forbid" if you prefer strict rejection of unknown fields
    ...
```

## Exercise
Implement the schemas, write parametrized tests for boundary values (0, 1, max, bad category, future date), and set `extra="forbid"` on input models. Compare Pydantic and Zod side-by-side for the same schema.

## Interview Q&A
- **Why separate request and response models?** Security (mass assignment/leaks), API stability, different required fields.
- **How do you do partial updates?** Optional model + `exclude_unset=True`.
- **Pydantic v1 vs v2?** v2 core in Rust (faster), `model_validate`, `model_dump`, `field_validator`, `ConfigDict`.
- **Where do you validate—Pydantic or DB?** Both: Pydantic for input/UX, DB constraints as the last line of defense.
