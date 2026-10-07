# Full-Stack 07 — Interview Questions, Behavioral Stories and Questions to Ask

## Technical quick-fire (full stack)
**Web fundamentals**
1. What happens when you type a URL and press Enter? (DNS → TCP/TLS → HTTP → server → response → parse HTML → CSS/JS → render; mention CDN, caching, HTTP/2.)
2. HTTP methods, status codes, headers (`Cache-Control`, `ETag`, `Content-Type`, `Authorization`, `Set-Cookie`).
3. Cookies vs localStorage vs sessionStorage; `HttpOnly`, `Secure`, `SameSite`.
4. CORS: what, why, preflight. Can the server "fix" CORS for non-browser clients? (CORS is browser enforcement.)
5. HTTP/1.1 vs HTTP/2 vs HTTP/3. WebSockets vs SSE vs long polling.
6. CSS: box model, Flexbox vs Grid, specificity, `rem` vs `px`, responsive design, `position`.
7. JS: closures, `this`, event loop, promises, prototypal inheritance, `==` vs `===`, hoisting, debounce vs throttle.
8. Web performance: Core Web Vitals, critical rendering path, lazy loading, caching strategy.
9. Accessibility basics.

**Backend / API**
10. REST principles, idempotency, pagination, versioning ([01](01-rest-api-design.md)).
11. AuthN vs AuthZ, JWT, OAuth2 flows, OIDC, PKCE ([02](02-authentication-jwt-oauth.md)).
12. SQL joins/window functions/indexes/transactions ([03](03-sql-and-nosql-modeling.md)).
13. SQL vs NoSQL choice; CAP; consistency models.
14. Caching strategies; invalidation; stampede.
15. Queues and async processing; at-least-once vs exactly-once; idempotent consumers.
16. Concurrency: race conditions, locks, optimistic vs pessimistic.
17. Security: OWASP Top 10, XSS, CSRF, SQLi, SSRF, BOLA.
18. Testing strategy ([05](05-testing-and-devops.md)).
19. CI/CD pipeline you'd design ([../aws/08](../aws/08-cicd-observability-and-cost.md)).
20. Docker vs VM; container basics; why multi-stage builds.

**Cloud/AI**
21. Lambda vs ECS; RDS vs DynamoDB; API Gateway features.
22. How would you add a Bedrock-powered feature safely and cheaply? ([../aws/07](../aws/07-bedrock-and-ai-integration.md))
23. Explain MCP and build a tool ([../mcp/](../mcp/)).

## Coding exercise types to expect
- **Array/string/hash-map problems** (two-sum, group anagrams, sliding window, merge intervals, LRU cache) in TS/Python/Go. Practice 20 easy/medium; talk through complexity.
- **Domain tasks**: "write `groupByCategory` and monthly totals", "parse CSV and dedupe", "paginate a list", "implement debounce".
- **Take-home/pair**: build a small CRUD API + React UI with tests in 60–90 minutes.

LRU cache sketch (Python):

```python
from collections import OrderedDict
class LRU:
    def __init__(self, cap: int): self.cap, self.d = cap, OrderedDict()
    def get(self, k):
        if k not in self.d: return None
        self.d.move_to_end(k); return self.d[k]
    def put(self, k, v):
        self.d[k] = v; self.d.move_to_end(k)
        if len(self.d) > self.cap: self.d.popitem(last=False)
```

## Behavioral: STAR stories to prepare (Situation, Task, Action, Result)
Prepare 6, reusable for many questions. Quantify results.
1. **Technical challenge / hard bug** you solved.
2. **Production incident**: detection, mitigation, root cause, prevention.
3. **Cost or performance win** (e.g. cut latency 40%, saved $2k/month).
4. **Disagreement** with a teammate/stakeholder and how you resolved it.
5. **Shipping under ambiguity / tight deadline**; scope trade-offs.
6. **Mentoring or code review** that raised quality.
Also: a failure and what you learned; learning a new tech fast (AI/MCP/Bedrock is a good example: this project).

"**Tell me about yourself**" (60–90 s): present (what you do) → past (relevant experience, 2 highlights) → future (why this role). Tie to React/TS/AWS/AI.

"**Why this role?**" Connect to the JD: full-stack with React + AWS, AI features (Bedrock/Azure AI Foundry), CI/CD ownership, collaboration.

## Communicating in a technical interview
- Think aloud; restate the problem; ask clarifying questions; propose an approach before coding; test with examples; discuss complexity and edge cases.
- If stuck: simplify, give a brute-force solution first, ask for a hint without apologizing.
- State trade-offs ("I chose X because…, the downside is…").
- Admit what you don't know and describe how you'd find out.

## Questions to ask them (pick 4–5)
**Team/project**
- What does the team look like (roles, size), and who would I work with most closely?
- What's the product/project and what stage is it in? What does success look like in 6 months?
- What does the architecture look like today (React app, API style, AWS services in use)?
**Engineering practice**
- How does code go from commit to production? How often do you deploy? What happens when a deploy fails?
- How do you handle code review, testing expectations, and tech debt?
- How do you monitor production and handle on-call/incidents?
**AI**
- Which AI use cases are live? How do you evaluate and version prompts/models? Which platforms (Bedrock/Azure AI Foundry) and why?
- Are you building or consuming MCP servers?
**Growth/culture**
- How are decisions made? How do you support learning (conferences, certifications)?
- What's the hardest problem the team is facing right now?
**Logistics** (if relevant): working hours/time zones, remote policy, next steps in the process.

## Final-week checklist
- [ ] Reread the JD; map each bullet to one concrete story or project from this repo
- [ ] Run through each `*-interview-questions.md` aloud
- [ ] Have the expense tracker running locally (React + one backend) for a demo/screen share
- [ ] Prepare a 2-minute architecture explanation of the project and the AWS design
- [ ] Practice one live-coding drill per topic (React, TS, API, SQL, Go/Python)
- [ ] Prepare environment: IDE, mic, camera, internet backup, water; arrive 10 min early
