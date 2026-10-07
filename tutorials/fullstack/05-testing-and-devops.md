# Full-Stack 05 — Testing Strategy, Git/GitHub Workflow and DevOps Tooling

## Why it matters
JD: "Proficiency in DevOps tools (GitHub Actions, SonarQube, GitHub)", "code reviews, bug fixing, troubleshooting". The pipeline specifics are in [../aws/08](../aws/08-cicd-observability-and-cost.md); this covers strategy and workflow.

## Testing pyramid (and where each tool fits)

```
        /\   E2E (Playwright)         few, slow, critical journeys
       /  \  Integration/API tests    real DB (Testcontainers), HTTP-level
      /____\ Unit tests               many, fast, pure logic
```

| Layer | React/TS | Python | Go |
|---|---|---|---|
| Unit | Vitest | pytest | `testing` + testify |
| Component | Testing Library + MSW | n/a | n/a |
| API/integration | Supertest | pytest + FastAPI `TestClient` + Postgres container | `httptest` + `testcontainers-go` |
| Contract | OpenAPI schema validation (Schemathesis/Dredd) | Schemathesis | same |
| E2E | Playwright | | |

Principles: test behavior, not implementation; deterministic (control time, randomness, network); isolate (each test sets up its data); fast feedback; coverage is a signal not a goal (gate on *new code* coverage).

### Integration test sample (FastAPI + real Postgres)

```python
# tests/test_expenses_api.py
def test_create_and_list_expense(client, auth_headers):
    r = client.post("/expenses", json={"amountCents": 1250, "currency": "USD", "category": "food", "date": "2026-10-01"}, headers=auth_headers)
    assert r.status_code == 201
    assert r.headers["location"].startswith("/expenses/")

    r = client.get("/expenses?from=2026-10-01&to=2026-10-31", headers=auth_headers)
    assert [e["amountCents"] for e in r.json()["items"]] == [1250]

def test_cannot_read_other_users_expense(client, auth_headers, other_users_expense_id):
    assert client.get(f"/expenses/{other_users_expense_id}", headers=auth_headers).status_code == 404

def test_requires_auth(client):
    assert client.get("/expenses").status_code == 401
```
Test the **negative and security cases** as seriously as the happy path.

### Test doubles
Dummy, stub, fake (in-memory repo), mock (asserts calls), spy. Prefer fakes over heavy mocking; mock only at boundaries (network, clock, Bedrock).

### Testing AI features
LLM output is non-deterministic: (1) unit-test the plumbing with a **stubbed model client**, (2) **golden-set evals** with assertions on structure and key facts (JSON schema valid, totals match, no hallucinated categories), (3) run evals in CI on prompt changes, (4) monitor in production with feedback signals.

## Git workflow
- **Trunk-based** (short-lived branches, feature flags, frequent merge to `main`) vs **GitFlow** (long-lived develop/release branches). Trunk-based pairs well with strong CI.
- Branch naming `feat/…`, `fix/…`; **Conventional Commits** (`feat: add budget endpoint`) enable changelogs/semver automation.
- **PR hygiene**: small, linked to an issue, description with what/why/how to test, screenshots, checklist; required reviews + status checks; squash-merge for linear history.
- Useful commands: `git rebase -i`, `git bisect` (find the commit that broke it), `git reflog` (recover), `git cherry-pick`, `git stash`, `git revert` (safe undo on shared branches) vs `reset` (rewrites history; local only).
- **Merge vs rebase**: merge preserves history; rebase makes it linear; never rebase shared/public branches.

## GitHub features to know
Branch protection / rulesets, CODEOWNERS, required status checks, environments with approvals and secrets, Dependabot (version + security updates), secret scanning/push protection, code scanning (CodeQL), reusable workflows, matrix builds, caching, `concurrency` groups to cancel superseded runs, OIDC to AWS, releases/tags, Actions artifacts.

```yaml
# .github/workflows/pr.yml (matrix + cache + concurrency)
name: pr
on: pull_request
concurrency: { group: pr-${{ github.ref }}, cancel-in-progress: true }
jobs:
  web:
    runs-on: ubuntu-latest
    strategy: { matrix: { node: [20, 22] } }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "${{ matrix.node }}", cache: npm, cache-dependency-path: web/package-lock.json }
      - run: npm ci && npm run lint && npm run typecheck && npm test -- --coverage
        working-directory: web
```

## SonarQube / quality gates
Static analysis categories: **bugs, vulnerabilities, security hotspots, code smells, duplications, coverage**. **Quality gate** conditions on *new code* (e.g. coverage ≥ 80%, duplication < 3%, no new blocker issues, security rating A). "Clean as you code": don't fail PRs for legacy debt, but never add new debt. Add a `sonar-project.properties`:

```properties
sonar.projectKey=expense-tracker
sonar.sources=web/src,api/app
sonar.tests=web/src,api/tests
sonar.test.inclusions=**/*.test.tsx,**/test_*.py
sonar.javascript.lcov.reportPaths=web/coverage/lcov.info
sonar.python.coverage.reportPaths=api/coverage.xml
```
Complement with: dependency scanning (`npm audit`, `pip-audit`, `govulncheck`, Dependabot), container scanning (Trivy/ECR scanning), secret scanning (gitleaks), IaC scanning (Checkov/cfn-nag).

## Docker & local environment
`docker compose` for local Postgres + API + MCP + web; same images in prod; `.env.example` committed, real secrets not.

```yaml
services:
  db:
    image: postgres:16
    environment: { POSTGRES_PASSWORD: dev, POSTGRES_DB: expenses }
    ports: ["5432:5432"]
    healthcheck: { test: ["CMD-SHELL", "pg_isready -U postgres"], interval: 5s, retries: 10 }
  api:
    build: ./api-fastapi
    environment: { DATABASE_URL: postgresql+psycopg://postgres:dev@db:5432/expenses }
    depends_on: { db: { condition: service_healthy } }
    ports: ["8000:8000"]
```

## Debugging and troubleshooting method (they ask "tell me about a hard bug")
1. **Reproduce** reliably (minimal case, specific input/env).
2. **Observe**: logs, traces, metrics, network tab, DB query logs; form hypotheses.
3. **Bisect** (git bisect, feature flags, disable half the pipeline).
4. **Fix the cause, add a regression test.**
5. **Postmortem**: what allowed it to ship? What detection was missing? Blameless.

Tools: browser devtools (Network, Performance, React Profiler), `curl -v`/httpie, Postman/Bruno, CloudWatch Logs Insights, X-Ray, `EXPLAIN`, debugger breakpoints (VS Code), `pprof` (Go), `py-spy`/`cProfile` (Python).

## Collaboration (soft-skill signals the JD names)
Clarify requirements early; write short design docs/ADRs; communicate trade-offs; estimate with ranges; give and receive code review feedback (be specific, kind, focus on the code); document (README, runbooks, OpenAPI); demo to stakeholders; pair program; own on-call/incident follow-ups.

## Exercise
Set up the monorepo CI with the matrix above, SonarQube (SonarCloud free tier for public repos), Dependabot, branch protection requiring checks, and a PR template. Introduce a failing test and a code smell and watch the gates stop the merge.

## Interview Q&A
- **Unit vs integration vs E2E?** Scope, speed, confidence trade-off (pyramid).
- **How do you test code that uses time/randomness/network?** Inject clocks/RNG, fake the boundary (MSW, stubs).
- **Rebase vs merge?** See above; never rewrite shared history.
- **What does your quality gate enforce and why on new code?** Prevents regression without blocking on legacy.
- **How do you review a PR?** Understand intent → tests → correctness/security → design/naming → performance → nits last (automate nits).
- **Flaky test policy?** Quarantine + fix root cause (timing/shared state), don't retry-until-green.
