# Full-Stack 08 — SOLID and Clean Architecture (the reference for every stack)

This is the **architecture contract** for the whole project. The per-stack tutorials apply it:
[fastapi/07](../fastapi/07-clean-architecture.md) · [gin/07](../gin/07-clean-architecture.md) · [react/08](../react/08-clean-architecture-frontend.md) · [angular/06](../angular/06-clean-architecture-frontend.md) · [typescript/07](../typescript/07-solid-in-typescript.md) · [mcp/07](../mcp/07-mcp-and-assistant-as-adapters.md).

## Why this project is the perfect case for it
The whole point of the repo is that **things are swappable**:

| We swap… | …without touching |
|---|---|
| React ↔ Angular | domain rules, API contract |
| FastAPI ↔ Gin | domain rules, front-ends, AI plane |
| DynamoDB ↔ Postgres | use cases, HTTP layer |
| Bedrock ↔ Azure AI Foundry ↔ a fake | assistant/insight logic |
| REST ↔ MCP (two *delivery mechanisms*) | business rules |

Clean architecture is the discipline that makes each of those a one-folder change. If you can say that sentence in an interview and point at the folders, you've answered "tell me about architecture".

## The Dependency Rule

```
        ┌──────────────────────────────────────────────────────────┐
        │ Frameworks & Drivers   FastAPI, Gin, boto3, DynamoDB,    │
        │                        Bedrock SDK, React, Angular, fetch│
        │  ┌────────────────────────────────────────────────────┐  │
        │  │ Interface Adapters  controllers, presenters/DTOs,  │  │
        │  │                     repository impls, MCP tools    │  │
        │  │  ┌──────────────────────────────────────────────┐  │  │
        │  │  │ Application (Use Cases)  + Ports (interfaces)│  │  │
        │  │  │  ┌────────────────────────────────────────┐  │  │  │
        │  │  │  │ Domain: entities, value objects, rules │  │  │  │
        │  │  │  └────────────────────────────────────────┘  │  │  │
        │  │  └──────────────────────────────────────────────┘  │  │
        │  └────────────────────────────────────────────────────┘  │
        └──────────────────────────────────────────────────────────┘
        Source-code dependencies point INWARD only.
```
**Nothing in an inner circle may import, name, or know about anything in an outer circle** (no `boto3` in a use case, no `gin.Context` in the domain, no `fetch` in the domain). Data crosses boundaries as plain structures (DTOs / domain objects), never framework types.

How does the use case call the database if it can't import it? **Dependency inversion**: the use case declares a *port* (an interface it needs), and an outer adapter implements it. At runtime, a **composition root** wires them.

## Layers mapped to the expense tracker

| Layer | Contains | Example | May depend on |
|---|---|---|---|
| **Domain** | Entities, value objects, domain services, domain errors. Pure code, no I/O | `Expense`, `Money`, `Category`, `summarize()`; "date cannot be in the future" | nothing |
| **Application** | Use cases (one per user intent) + **ports** (interfaces for what they need) | `CreateExpense`, `ListExpenses`, `MonthlySummary`, `GenerateInsight`; ports `ExpenseRepository`, `Clock`, `IdGenerator`, `InsightGenerator` | domain |
| **Interface adapters** | Convert between outside world and use cases; implement ports | HTTP controllers + request/response schemas, DynamoDB repository, Bedrock `InsightGenerator`, MCP tools, error→HTTP mapping | application, domain |
| **Frameworks & drivers** | The actual tools and wiring | FastAPI/Gin app, boto3/AWS SDK, Lambda Web Adapter, CDK | everything (it's the outermost) |

Rules of thumb:
- **Entities** hold rules that are true regardless of the app ("amount must be positive").
- **Use cases** hold application-specific rules/orchestration ("a user can only delete their own expense", "generate an insight from this month's totals").
- **Adapters** hold translation and nothing else: no business decisions.

## SOLID, applied (not recited)

### S — Single Responsibility
*A module has one reason to change.*
Before: one FastAPI handler validates JSON, checks auth, builds a DynamoDB key, calls Bedrock, formats the HTTP response. Five reasons to change.
After: controller (HTTP translation) → use case (one intent) → repository (persistence) → domain object (rule). Changing the table design touches one file.

### O — Open/Closed
*Open for extension, closed for modification.*
Add Azure AI Foundry as an insight provider by writing a new `InsightGenerator` adapter and changing **one line** in the composition root, with no edit to `GenerateInsight`. Add a new category rule by adding a rule object, not another `if` branch in a 200-line function. Tools for achieving it: polymorphism/strategy behind a port, registries/maps of handlers, discriminated unions with exhaustive checks.

### L — Liskov Substitution
*Any implementation must be usable wherever the abstraction is expected, without surprises.*
`InMemoryExpenseRepo` and `DynamoDbExpenseRepo` must behave the same: same ordering, same "not found" behavior, same "other user's data is invisible". **Prove it with one shared contract test suite run against every implementation.** (Each stack tutorial includes it.) A repo that returns an expense for the wrong user violates LSP *and* security.

### I — Interface Segregation
*Clients shouldn't depend on methods they don't use.*
`CreateExpense` needs `add`; it shouldn't depend on a fat `Repository` with 15 methods. Split ports by role (`ExpenseReader`, `ExpenseWriter`); a concrete class can implement both. In Go this is the idiom (small consumer-defined interfaces); in TypeScript use `Pick<>` or small interfaces; in Python use small `Protocol`s.

### D — Dependency Inversion
*High-level policy shouldn't depend on low-level detail; both depend on abstractions.*
`CreateExpense` (policy) → `ExpenseWriter` (abstraction) ← `DynamoDbExpenseRepo` (detail). The arrow from detail to abstraction points *toward* the application. Constructor injection + composition root is the mechanism. **DIP is the engine that makes the dependency rule possible.**

## Anatomy of a use case (the template every stack follows)

```
Input DTO  ──►  UseCase.execute(input)  ──►  Output (domain object or output DTO)
                    │ uses ports only
                    ├─ ExpenseWriter.add(expense)
                    ├─ Clock.today()
                    └─ IdGenerator.new_id()
```
- One use case = one class/function = one user intent. Name it as a verb phrase: `CreateExpense`, `ListExpenses`.
- Input is a plain data object (not a Pydantic/Gin binding type; those belong to the HTTP adapter).
- It raises/returns **domain/application errors** (`InvalidExpense`, `ExpenseNotFound`), never `HTTPException` or status codes.
- Contains orchestration and application rules; delegates invariants to domain objects.
- Trivially testable with fakes: no framework, no network, no AWS.

## Ports and adapters in this project

| Port (in application layer) | Adapters (outside) |
|---|---|
| `ExpenseReader` / `ExpenseWriter` | `DynamoDbExpenseRepo`, `PostgresExpenseRepo` (future), `InMemoryExpenseRepo` (tests) |
| `Clock`, `IdGenerator` | `SystemClock`, `UlidGenerator`; `FixedClock`, `SequentialIds` (tests) |
| `ReceiptStorage` | `S3ReceiptStorage` (presigned URLs), `FakeReceiptStorage` |
| `InsightGenerator` / `LlmClient` | `BedrockLlmClient`, `AzureFoundryLlmClient` (future), `FakeLlm` (tests/evals) |
| `ToolGateway` (assistant → tools) | `McpToolGateway`, `InProcessToolGateway` |
| `ExpensesGateway` (MCP server → REST) | `HttpExpensesGateway`; or direct use cases when co-hosted |
| Front-end `ExpenseGateway` | `HttpExpenseGateway` (fetch/HttpClient + DTO mapping), `FakeExpenseGateway` |

**Driving (inbound) adapters** call use cases: HTTP controllers, MCP tools, CLI, queue consumers, React hooks. **Driven (outbound) adapters** are called by use cases through ports: repositories, LLMs, storage. REST and MCP are two inbound adapters over the *same* use cases.

## Composition root
The one place that knows concrete types and wires them: `container.py` / `cmd/api/main.go` / `app.config.ts` / `composition.ts`. Everything else receives dependencies (constructor injection). Tests build a different root with fakes. No service locators, no global singletons buried in modules, no `import boto3` inside use cases.

## Boundaries are data, not frameworks
- Request schema (Pydantic / Gin binding struct / Zod DTO) ≠ use-case input ≠ domain entity ≠ DB item ≠ response schema. Mapping is boilerplate by design; it's what lets each side change independently. Keep mappers small and explicit.
- Never return ORM/Dynamo items or SDK types from a port. A repository returns domain objects.
- Errors: domain errors are translated to HTTP statuses/JSON-RPC errors **only** in the adapter.

## Testing pyramid, aligned to the layers

| Layer | Test type | Needs |
|---|---|---|
| Domain | pure unit tests, property tests | nothing |
| Use cases | unit tests with fake ports | fakes only (milliseconds) |
| Repository adapters | **contract tests** shared by all implementations; DynamoDB Local for the real one | DynamoDB Local |
| HTTP/MCP adapters | request-level tests with the app built from a fake-backed container | test client |
| Whole system | smoke tests, evals | deployed stack ([../infra/08](../infra/08-operations-observability-cost-teardown.md)) |

Most logic is tested at the use-case level, the fastest and most valuable layer.

## Enforce the dependency rule automatically
Architecture decays unless a machine checks it:
- **Python**: `import-linter` contracts (`layers` contract: `interface > adapters > application > domain`) in CI.
- **Go**: `depguard`/`golangci-lint` rules, or a test using `go list -deps`; package layout + `internal/` helps.
- **TypeScript**: ESLint `no-restricted-imports` per folder (or `eslint-plugin-boundaries` / Nx module boundaries); `dependency-cruiser`.
- Add these to the CI pipeline ([../infra/07](../infra/07-cicd-github-actions.md)) next to tests and SonarQube.

## Pragmatism: how not to over-engineer (interviewers probe this)
- Clean architecture has a **cost**: more files, mapping code, indirection. Pay it where change/volatility and testing value are high (business rules, persistence, AI providers), not for trivial pass-through code.
- **Rule of three**: introduce an abstraction when you have (or can name) a second implementation or a real test need. Here we do have them (in-memory/Dynamo, fake/Bedrock, REST/MCP).
- Don't create a port for every library (a `StringUtils` interface is noise). Ports model *capabilities your use cases need*.
- Avoid the **anemic domain model** (entities that are bags of fields with all logic in services). Put invariants in the entity/value object.
- Avoid **leaky ports** (a repository method that returns a DynamoDB `LastEvaluatedKey` type): use an opaque cursor string.
- For genuine throwaway CRUD, a thin vertical slice is fine. Say so, and explain the trade-off.
- Related ideas to name-drop correctly: **Hexagonal architecture** (ports & adapters; the same idea as clean), **Onion architecture**, **DDD** (entities, value objects, aggregates, bounded contexts), **CQRS** (separate read models, useful for reports), **vertical slice architecture** (organize by feature, each slice may have its own layers).

## Folder conventions across the repo

```
api-fastapi/app/{domain,application,adapters,interface,container.py,main.py}
api-gin/internal/{domain,app,adapters/{ddb,memory,httpapi}}  + cmd/api/main.go
web-react/src/{domain,application,infrastructure,presentation,app}
web-angular/src/app/{domain,application,infrastructure,presentation}
assistant/{domain|core, adapters, main}      mcp-server/{application, adapters, main}
```
Each stack uses its idioms (Go: consumer-side interfaces and `internal/`; TypeScript: structural typing; Python: Protocols), but the **layer names and the dependency direction are identical**, so someone fluent in one stack can navigate the others.

## Worked example, end to end: "Create expense"

```
POST /expenses  (Lambda/ECS)
 └─ interface/http (controller): parse JSON → CreateExpenseRequest → map to CreateExpenseInput, identity from the gateway context
     └─ application.CreateExpense.execute(input)
         ├─ clock.today()                           (port)
         ├─ ids.new_id()                            (port)
         ├─ domain.Expense.create(...)              (entity enforces invariants → InvalidExpense)
         └─ writer.add(expense)                     (port)
              └─ adapters.DynamoDbExpenseRepo.add → put_item(PK=USER#sub, SK=EXP#<ulid>, …)
     ◄─ returns Expense (domain)
 └─ controller maps Expense → ExpenseResponse (201 + Location); DomainError → problem+json (422/404)
```
Same use case is reachable from an MCP tool `add_expense` ([../mcp/07](../mcp/07-mcp-and-assistant-as-adapters.md)) without copying any rule.

## Exercises
1. Take any handler from earlier tutorials and mark every line as domain / application / adapter / framework. Move the misplaced lines.
2. Write the import-linter (or ESLint) rule that makes `domain` unable to import `boto3`/`react`, then break it on purpose and watch CI fail.
3. Swap the repository for the in-memory one using only the composition root and run the same use-case tests.

## Interview Q&A
- **What is the dependency rule?** Source-code dependencies point inward; inner layers know nothing of outer ones.
- **How does a use case talk to the database without depending on it?** Port (interface) defined in the application layer; adapter implements it; wired in the composition root (DIP).
- **Difference between entity and use case?** Enterprise-wide invariants vs application-specific orchestration.
- **How do you test a use case?** Fake ports, no I/O.
- **How do you know an implementation honors the port (LSP)?** A shared contract test suite run against every adapter.
- **When is clean architecture overkill?** Small throwaway CRUD, prototypes with no volatility; mention the cost/benefit honestly.
- **Where do validation rules live?** Syntactic/shape validation at the adapter (schema); business invariants in the domain; authorization/ownership in the use case + repository scoping.
- **Hexagonal vs clean vs onion?** Same core idea (domain at center, dependencies inward, ports/adapters) with different diagrams and vocabulary.
- **How does this help swap React for Angular?** Both talk to the same API contract; each has its own domain/application/infrastructure layers so the logic ports across, and only the presentation layer is rewritten.
