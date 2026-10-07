# Interview Prep Tutorials — Full Stack Engineer (AWS / React / TypeScript / MCP)

> **The code these tutorials describe now exists at the repo root** ([README](../README.md)): `web-react`, `web-angular`, `api-fastapi`, `api-gin`, `mcp-server`, `assistant`, `infra`. Run `make test` to see it all pass. Where a tutorial snippet and the code differ, the code (which is tested) wins; the tutorials call out the differences.

Every tutorial uses the same running example, the **Expense Tracker**, so concepts stack on each other instead of being isolated toy demos.

## Domain model (used everywhere)

```
User     { id, email }
Expense  { id, userId, amountCents, currency, category, description, date, receiptKey?, createdAt }
```

REST surface:

| Method | Path | Purpose |
|---|---|---|
| POST | `/expenses` | create |
| GET | `/expenses?from=&to=&category=&cursor=` | list (filter, cursor paginated) |
| GET | `/expenses/{id}` | read |
| PATCH | `/expenses/{id}` | partial update |
| DELETE | `/expenses/{id}` | delete |
| GET | `/reports/summary?month=2026-10` | totals by category |

## Map: interview topic → folder

| Priority | Interview topic | Folder | Notes |
|---|---|---|---|
| MUST | Amazon Web Services | [aws/](aws/) | API Gateway, Lambda, S3, ECS, RDS, Cognito, Bedrock, CI/CD |
| MUST | Full-Stack Development | [fullstack/](fullstack/) | REST, auth (OAuth/JWT), SQL/NoSQL, architecture, system design |
| MUST | Model Context Protocol | [mcp/](mcp/) | Build servers (TS + Python), clients, Bedrock integration |
| MUST | ReactJS | [react/](react/) | Hooks, state, data fetching, performance, testing |
| MUST | TypeScript | [typescript/](typescript/) | Type system, generics, narrowing, React/Node usage |
| JD | Python backend | [fastapi/](fastapi/) | JD says "Python or Go" |
| NICE | Go Language | [gin/](gin/) | Go fundamentals + Gin REST API |
| NICE | Angular | [angular/](angular/) | Modern Angular (standalone, signals), React comparison |
| ARCH | SOLID and Clean Architecture | [fullstack/08](fullstack/08-solid-and-clean-architecture.md) | The architecture contract; applied in `fastapi/07`, `gin/07`, `react/08`, `angular/06`, `typescript/07`, `mcp/07` |
| DEPLOY | Infrastructure (CDK) | [infra/](infra/) | Deploy `react-fastapi`, `angular-fastapi`, `react-gin` (DynamoDB + Bedrock + MCP) |

Each folder ends with an interview-questions file (Angular: `05-react-vs-angular-and-interview.md`): drill those out loud.

## Suggested study order

1. `typescript/` 01–03 (everything else is typed)
2. `react/` 01–05
3. `fullstack/` 01–03 (REST + auth + data), then `fullstack/08` + `typescript/07` (SOLID and clean architecture: build every later stack on this)
4. `aws/` 01–06
5. `mcp/` 01–04
6. `aws/07` Bedrock + `mcp/04` (the AI part of the JD)
7. `fastapi/` and `gin/` (pick one to be fluent, skim the other)
8. `fullstack/06` system design, then `angular/`
9. `infra/` 01–09: deploy the three variants end to end (after the technology folders)
10. All the interview-question files, out loud, timed

## How to practice (it's a technical interview)

- **Type the code**, do not read it. Each tutorial has an "Exercise" section.
- **Explain trade-offs aloud**: Lambda vs ECS, SQL vs DynamoDB, JWT vs session, REST vs GraphQL. Interviewers grade the reasoning more than the answer.
- **Know one thing deeply per topic**, e.g. one full request path: browser → CloudFront → API Gateway → Lambda → RDS, including auth, logging and failure modes.
- Prepare **questions for them** (see [fullstack/07-interview-questions.md](fullstack/07-interview-questions.md)).

> Note: AWS service names, console flows and model IDs change. Where a snippet pins a version or model ID, verify it against the current docs before relying on it.
