# Infra 09 — Interview Questions and the 5-Minute Walkthrough

## The walkthrough (practice this; it's your best "tell me about a project you built")

> "I built an expense tracker as a practice platform and deployed it in three interchangeable versions: React or Angular on the front, FastAPI or Gin in the middle, DynamoDB underneath, all with Bedrock and MCP features. It's one CDK app where a *variant* is just a config object, so each variant gets its own isolated stacks.
>
> The browser hits CloudFront, which serves the SPA from a private S3 bucket and proxies `/api/*` to an API Gateway HTTP API. A Cognito JWT authorizer validates the access token before any code runs. Behind it, the CRUD service is a container on Lambda via the Lambda Web Adapter, so the same Dockerfile pattern works for Python and Go. The data is a DynamoDB single table keyed by the Cognito `sub`, which makes cross-tenant access structurally impossible.
>
> The AI side is a separate plane shared by all variants: an MCP server that's a thin adapter over the REST API and an assistant service that runs a Bedrock Converse tool-use loop. Every hop forwards the user's token, so the model can only do what the user could do. I bound cost with token and turn caps, reserved concurrency, gateway throttling and budgets, and I gate changes with agent evals in CI.
>
> Deploys are GitHub Actions with OIDC, a matrix over the variants, `cdk diff` on PRs, and an authenticated smoke test after each deploy. I measured cold starts, latency and bundle sizes across variants and wrote down what I'd ship and why."

Be ready to go one level deeper on any sentence.

## Architecture questions
1. **Why Lambda + LWA instead of ECS?** Spiky/low traffic → pay per use, no idle cost, no VPC/NAT. Where would you switch? Streaming responses beyond 30 s, long-lived connections, steady high traffic, MCP features needing server-initiated streams.
2. **Why one table in DynamoDB?** Access patterns known up front; single-digit-ms key access; no connection management for Lambda; on-demand billing. What would make you add Postgres? Ad-hoc reporting, joins, constraints ([../fullstack/03](../fullstack/03-sql-and-nosql-modeling.md)).
3. **Why is a ULID sort key better here than a date-based key?** Direct `GetItem` by id; date range via GSI. Trade-off: GSI storage/write cost.
4. **Walk me through authentication on a single request.** Access token from Hosted UI (Auth Code + PKCE) → CloudFront forwards `Authorization` → API Gateway validates signature/issuer/`client_id`/expiry → Lambda gets verified claims in the request context → repository key `USER#<sub>`.
5. **What if someone invokes the Lambda directly with a forged context header?** They need `lambda:InvokeFunction` IAM permission. Resource policy only allows this API. No public function URL.
6. **Why does MCP forward the user's token instead of using a service role?** Confused-deputy avoidance; authorization logic stays in one place. Caveat: only acceptable when the downstream is the token's audience.
7. **Why does the assistant call `/mcp` over HTTP rather than in-process?** Demonstrates and tests the real MCP boundary other hosts will use; cost is latency/hops. Trade-off discussion shows judgment.
8. **How would you add streaming tokens to the UI?** Lambda Function URL with response streaming (needs its own auth: Lambda URL IAM + signed requests or an ECS service behind ALB/Cognito), SSE to the SPA; API Gateway HTTP APIs buffer.
9. **How do you version the AI plane safely?** Tool schemas are an API: additive changes, snapshot tests of `tools/list`, evals before release.
10. **How would you make this multi-region?** Route 53 latency/failover, DynamoDB Global Tables, Cognito per region or federated, S3 CRR, Bedrock regional availability, IaC stamped per region; and say you'd need a real RTO/RPO to justify the cost.

## IaC / CDK questions
- What happens on `cdk deploy`? (synth → CloudFormation template + assets → bootstrap roles publish assets → changeset → execution, rollback on failure)
- What is bootstrapping and why is it required?
- How did you handle the circular dependency between Cognito callbacks and the CloudFront domain? (two-pass deploy; alternative: custom domain known in advance, or a custom resource)
- How do you avoid replacing a stateful resource accidentally? (`cdk diff`, logical ID stability, `RETAIN`, deletion protection, avoid changing construct ids/names)
- CDK vs Terraform vs SAM vs Serverless Framework?
- How do you test infrastructure? (`assertions` module fine-grained tests, snapshot tests, `cdk-nag`, deploy to an ephemeral stage)

```ts
// infra/test/data-stack.test.ts
import { App } from "aws-cdk-lib";
import { Template } from "aws-cdk-lib/assertions";
import { DataStack } from "../lib/data-stack";
import { VARIANTS } from "../lib/config";

test("table has GSI1 and TTL, and the bucket is private", () => {
  const stack = new DataStack(new App(), "t", { variant: VARIANTS["react-fastapi"], stage: "dev", prefix: "x" });
  const t = Template.fromStack(stack);
  t.hasResourceProperties("AWS::DynamoDB::Table", {
    TimeToLiveSpecification: { AttributeName: "ttl", Enabled: true },
    GlobalSecondaryIndexes: [{ IndexName: "GSI1" }],
  });
  t.hasResourceProperties("AWS::S3::Bucket", {
    PublicAccessBlockConfiguration: { BlockPublicAcls: true, RestrictPublicBuckets: true },
  });
});
```

## Scenario drills
1. **"The assistant sometimes returns 504."** The gateway limit (30 s) was exceeded: tool turns × (Bedrock latency + two Lambda hops). Look at X-Ray, cut turns, shrink prompts/tool output, pick a faster model, or move to async job + polling / streaming.
2. **"A user reports seeing another user's expenses."** Treat as Sev-1: check key construction (`PK=USER#sub`), whether any code path takes the user id from input, authorizer config; add the isolation test to the smoke test; audit logs for cross-access; rotate nothing, since no secrets are involved, but invalidate sessions if necessary.
3. **"Bedrock bill spiked."** Cost Explorer by service/tag, CloudWatch Bedrock token metrics per hour, check the concurrency/throttle limits, look for a loop or abuse by a single `sub`, add per-user quotas (DynamoDB counter with TTL).
4. **"Cold starts hurt the first request."** Measure `@initDuration`; slim the image, lazy-import heavy libs, choose Go for latency-critical paths, or use provisioned concurrency on the API function only.
5. **"Marketing wants a custom domain and WAF."** ACM cert in `us-east-1`, Route 53 alias to CloudFront, WAF WebACL (rate-based + managed rules) on the distribution; this also removes the two-pass Cognito trick.
6. **"Add a fourth variant: Angular + Gin."** One entry in `VARIANTS`; no other code changes. Say why that's the payoff of designing around the REST contract.
7. **"Move the database to Postgres."** Only `api-*` repositories and the data stack change (RDS/Aurora Serverless + RDS Proxy + VPC); the AI plane, auth, CDN and front-ends are untouched. Quantify what you'd lose (zero idle cost, no VPC).

## Rapid-fire
- Lambda timeout vs API Gateway timeout? (up to 15 min vs 30 s for HTTP API)
- What does `reservedConcurrentExecutions` do? (reserves and caps; here, a cost ceiling)
- Why `PAY_PER_REQUEST`? When provisioned? (unpredictable/low traffic vs steady, predictable high traffic)
- Why OAC? Why not make the bucket public? (private origin, only CloudFront can read)
- What's in an HTTP API JWT authorizer check? (signature via JWKS from the issuer, `iss`, `aud`/`client_id`, `exp`/`nbf`, optional scopes)
- Why arm64? (Graviton price/performance; images must be built for it)
- Why not cache `/api/*` in CloudFront? (per-user data; use `Cache-Control` + keyed cache only for public endpoints)
- What's a stack output vs export? (readable value vs importable cross-stack reference)

## Questions to ask the interviewer
- How do you structure IaC and promotion across environments (CDK/Terraform, per-env accounts)?
- How are AI features evaluated and rolled back? Where do prompts live?
- What's your approach to cost visibility for LLM usage?
- Which parts of the platform are serverless vs containers, and why?
