# FastAPI 03 — PostgreSQL with SQLAlchemy 2.0 and Alembic

## Setup

```bash
docker run -d --name pg -e POSTGRES_PASSWORD=dev -e POSTGRES_DB=expenses -p 5432:5432 postgres:16
export DATABASE_URL=postgresql+psycopg://postgres:dev@localhost:5432/expenses
```

## ORM models (SQLAlchemy 2.0 typed style)

```python
# app/models.py
import uuid
from datetime import date, datetime
from sqlalchemy import BigInteger, CheckConstraint, Date, ForeignKey, Index, String, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    email: Mapped[str] = mapped_column(String(320), unique=True)
    cognito_sub: Mapped[str] = mapped_column(String(64), unique=True, index=True)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    expenses: Mapped[list["Expense"]] = relationship(back_populates="user", cascade="all, delete-orphan")

class Expense(Base):
    __tablename__ = "expenses"
    __table_args__ = (
        CheckConstraint("amount_cents > 0", name="ck_amount_positive"),
        Index("idx_expenses_user_date", "user_id", "spent_on", "id"),
    )
    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    user_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("users.id", ondelete="CASCADE"))
    amount_cents: Mapped[int] = mapped_column(BigInteger)
    currency: Mapped[str] = mapped_column(String(3), default="USD")
    category: Mapped[str] = mapped_column(String(50))
    description: Mapped[str | None] = mapped_column(String(200))
    spent_on: Mapped[date] = mapped_column(Date)
    receipt_key: Mapped[str | None]
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    version: Mapped[int] = mapped_column(default=1)               # optimistic locking
    user: Mapped[User] = relationship(back_populates="expenses")
```

## Engine, session, and the DB dependency

```python
# app/db.py
from collections.abc import Iterator
from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker
from app.config import settings

engine = create_engine(settings.database_url, pool_size=5, max_overflow=5, pool_pre_ping=True)
SessionLocal = sessionmaker(engine, expire_on_commit=False)

def get_db() -> Iterator[Session]:
    with SessionLocal() as session:
        yield session            # request-scoped; closed after the response
```
`pool_pre_ping` revives dead connections (RDS failover); keep `pool_size × instances` below the DB's `max_connections` (or use RDS Proxy). With Lambda use a tiny pool (1–2) and RDS Proxy.

> **Clean-architecture note:** in the layered design ([07](07-clean-architecture.md)) this class is an **adapter** implementing the `ExpenseReader`/`ExpenseWriter` ports and returning *domain* `Expense` objects (map ORM rows ↔ domain in the adapter). The ORM model never leaves `adapters/persistence/`.

## Repository with ownership built in

```python
# app/repositories/expenses.py
import base64, json
from datetime import date
from uuid import UUID
from sqlalchemy import select, tuple_
from sqlalchemy.orm import Session
from app.models import Expense

def encode_cursor(e: Expense) -> str:
    return base64.urlsafe_b64encode(json.dumps([e.spent_on.isoformat(), str(e.id)]).encode()).decode()

def decode_cursor(c: str) -> tuple[date, UUID]:
    d, i = json.loads(base64.urlsafe_b64decode(c.encode()))
    return date.fromisoformat(d), UUID(i)

class ExpenseRepo:
    def __init__(self, db: Session, user_id: UUID):
        self.db, self.user_id = db, user_id               # every query is scoped to the caller

    def get(self, expense_id: UUID) -> Expense | None:
        return self.db.scalar(select(Expense).where(Expense.id == expense_id, Expense.user_id == self.user_id))

    def list(self, start: date | None, end: date | None, category: str | None, limit: int, cursor: str | None):
        q = select(Expense).where(Expense.user_id == self.user_id)
        if start: q = q.where(Expense.spent_on >= start)
        if end: q = q.where(Expense.spent_on <= end)
        if category: q = q.where(Expense.category == category)
        if cursor:
            d, i = decode_cursor(cursor)
            q = q.where(tuple_(Expense.spent_on, Expense.id) < (d, i))      # keyset pagination
        q = q.order_by(Expense.spent_on.desc(), Expense.id.desc()).limit(limit + 1)
        rows = list(self.db.scalars(q))
        next_cursor = encode_cursor(rows[limit - 1]) if len(rows) > limit else None
        return rows[:limit], next_cursor

    def add(self, **data) -> Expense:
        e = Expense(user_id=self.user_id, **data)
        self.db.add(e)
        self.db.flush()                                    # get defaults without committing
        return e

    def monthly_totals(self, start: date, end: date):
        from sqlalchemy import func
        q = (select(Expense.category, func.sum(Expense.amount_cents), func.count())
             .where(Expense.user_id == self.user_id, Expense.spent_on >= start, Expense.spent_on < end)
             .group_by(Expense.category).order_by(func.sum(Expense.amount_cents).desc()))
        return self.db.execute(q).all()
```

Wire it with dependencies (transaction boundary at the request):

```python
def get_repo(db: Session = Depends(get_db), user_id: UUID = Depends(current_user_uuid)) -> ExpenseRepo:
    return ExpenseRepo(db, user_id)

@router.post("", response_model=ExpenseOut, status_code=201)
def create_expense(body: ExpenseCreate, response: Response, repo: ExpenseRepo = Depends(get_repo), db: Session = Depends(get_db)):
    e = repo.add(**body.model_dump())
    db.commit()
    response.headers["Location"] = f"/expenses/{e.id}"
    return e
```

## N+1 and loading strategies

```python
from sqlalchemy.orm import selectinload
db.scalars(select(User).options(selectinload(User.expenses)))     # 2 queries instead of N+1
```
Log SQL with `echo=True` in dev to catch N+1. Use `select()` with only needed columns for reports.

## Alembic migrations

```bash
alembic init alembic
# alembic/env.py: set target_metadata = Base.metadata and sqlalchemy.url from settings
alembic revision --autogenerate -m "create expenses"
alembic upgrade head
alembic downgrade -1
```
Always **review autogenerated migrations** (they miss some changes). Run `alembic upgrade head` as a pre-deploy job/ECS one-off task, not inside every app instance start. Zero-downtime = expand/contract.

## Async variant (when to bother)
`create_async_engine("postgresql+asyncpg://…")`, `AsyncSession`, `await session.scalars(…)`. Worth it for high-concurrency I/O-bound services on ECS; for Lambda or modest traffic the sync version is simpler and plenty fast.

## Testing against real Postgres

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def engine():
    with PostgresContainer("postgres:16") as pg:
        eng = create_engine(pg.get_connection_url().replace("psycopg2", "psycopg"))
        Base.metadata.create_all(eng)
        yield eng

@pytest.fixture
def db(engine):
    conn = engine.connect(); tx = conn.begin()
    session = Session(bind=conn)
    yield session
    session.close(); tx.rollback(); conn.close()          # isolation per test

@pytest.fixture
def client(db):
    app.dependency_overrides[get_db] = lambda: db
    app.dependency_overrides[current_user_id] = lambda: "test-user-sub"
    yield TestClient(app)
    app.dependency_overrides.clear()
```

## Exercise
Implement the repo + endpoints with keyset pagination; write a test creating 120 expenses and walking all pages with `limit=50` asserting no duplicates/skips; run `EXPLAIN ANALYZE` on the list query and the monthly report.

## Interview Q&A
- **Session lifecycle?** One session per request; commit/rollback at the boundary; close in `finally`.
- **Lazy vs eager loading?** Lazy triggers extra queries (N+1); choose `selectinload`/`joinedload` consciously.
- **How do you prevent lost updates?** `version` column with `WHERE version = :v`, or `with_for_update()`.
- **How do you run migrations in ECS?** One-off task / pipeline step before rolling the service, backward compatible.
- **SQL injection and ORMs?** Bound parameters; raw SQL via `text()` with params only.
