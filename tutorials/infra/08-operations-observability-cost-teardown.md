# Infra 08 — Smoke Tests, Observability, Cost Control and Teardown

## 1. The authenticated smoke test: `infra/scripts/smoke-test.sh`

Exercises the whole path: CloudFront → authorizer → API → DynamoDB, plus MCP and the Bedrock assistant. Works for any variant, which makes it the **conformance test** for "interchangeable back-ends".

```bash
#!/usr/bin/env bash
# usage: ./scripts/smoke-test.sh react-fastapi [dev]      (needs jq, curl, aws cli)
set -euo pipefail
VARIANT=${1:?variant}; STAGE=${2:-dev}; P="expense-${VARIANT}-${STAGE}"
out() { aws cloudformation describe-stacks --stack-name "$1" --query "Stacks[0].Outputs[?OutputKey=='$2'].OutputValue" --output text; }

URL=$(out "$P-web" WebUrl); POOL=$(out "$P-auth" UserPoolId); CLIENT=$(out "$P-auth" ClientId)
EMAIL="smoke-$(date +%s)@example.com"; PASS='Smoke-Test-12345'
TODAY=$(date +%F); MONTH=$(date +%Y-%m)

aws cognito-idp admin-create-user --user-pool-id "$POOL" --username "$EMAIL" --message-action SUPPRESS >/dev/null
trap 'aws cognito-idp admin-delete-user --user-pool-id "$POOL" --username "$EMAIL" >/dev/null 2>&1 || true' EXIT
aws cognito-idp admin-set-user-password --user-pool-id "$POOL" --username "$EMAIL" --password "$PASS" --permanent
TOKEN=$(aws cognito-idp admin-initiate-auth --user-pool-id "$POOL" --client-id "$CLIENT" \
  --auth-flow ADMIN_USER_PASSWORD_AUTH --auth-parameters "USERNAME=$EMAIL,PASSWORD=$PASS" \
  --query AuthenticationResult.AccessToken --output text)
H="Authorization: Bearer $TOKEN"
ok() { echo "✔ $1"; }

# 1. public health + SPA shell
curl -fsS "$URL/api/health" >/dev/null && ok "health via CloudFront"
curl -fsS "$URL/some/deep/link" | grep -qi "<html" && ok "SPA fallback serves index.html"
curl -fsS "$URL/config.json" | jq -e '.userPoolId and .apiBase' >/dev/null && ok "runtime config.json"

# 2. auth is enforced at the gateway
[ "$(curl -s -o /dev/null -w '%{http_code}' "$URL/api/expenses")" = 401 ] && ok "401 without token"

# 3. CRUD
ID=$(curl -fsS -X POST "$URL/api/expenses" -H "$H" -H 'content-type: application/json' \
  -d "{\"amountCents\":1250,\"currency\":\"USD\",\"category\":\"food\",\"date\":\"$TODAY\",\"description\":\"smoke lunch\"}" | jq -r .id)
[ -n "$ID" ] && ok "create → $ID"
curl -fsS "$URL/api/expenses/$ID" -H "$H" | jq -e ".id==\"$ID\"" >/dev/null && ok "get"
curl -fsS "$URL/api/expenses?from=$MONTH-01&to=$MONTH-31" -H "$H" | jq -e '.items|length>=1' >/dev/null && ok "list"
curl -fsS "$URL/api/reports/summary?month=$MONTH" -H "$H" | jq -e . >/dev/null && ok "summary"

# 4. MCP over Streamable HTTP (stateless JSON responses)
MCP_H=(-H "$H" -H 'content-type: application/json' -H 'accept: application/json, text/event-stream')
curl -fsS "$URL/api/mcp" "${MCP_H[@]}" -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"smoke","version":"1"}}}' \
  | jq -e '.result.serverInfo.name' >/dev/null && ok "mcp initialize"
curl -fsS "$URL/api/mcp" "${MCP_H[@]}" -d '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' \
  | jq -e '.result.tools|map(.name)|contains(["list_expenses","monthly_summary"])' >/dev/null && ok "mcp tools/list"

# 5. Bedrock assistant using MCP tools as this user
RESP=$(curl -fsS -X POST "$URL/api/assistant" -H "$H" -H 'content-type: application/json' \
  -d "{\"message\":\"How much did I spend on food in $MONTH?\"}")
echo "$RESP" | jq -e '.answer|length>0' >/dev/null && ok "assistant answered"
echo "$RESP" | jq -e '[.toolCalls[].name]|index("monthly_summary") or index("list_expenses")' >/dev/null && ok "assistant used an MCP tool"

# 6. isolation: a second user must not see the first user's expense (404)
# (create user B the same way and GET /expenses/$ID expecting 404 — extend as an exercise)

curl -fsS -X DELETE "$URL/api/expenses/$ID" -H "$H" -o /dev/null && ok "delete"
echo "ALL CHECKS PASSED for $P"
```
(An MCP server in stateless mode may reply as JSON or SSE depending on the SDK version/options. If `jq` fails on an `event:`/`data:` response, strip the SSE framing with `sed -n 's/^data: //p'` first.)

## 2. Observability

### Logs
Each Lambda writes to `/aws/lambda/<function>` with a two-week retention (set in `webFunction`). Emit **JSON logs with the request ID** (LWA forwards `x-amzn-lambda-context`/`x-amzn-request-context`, and API Gateway puts `requestId` in the context). Useful Logs Insights queries:

```
# slowest invocations and cold starts
filter @type = "REPORT"
| stats count() as n, avg(@duration) as avgMs, pct(@duration, 95) as p95Ms,
        max(@initDuration) as maxInitMs, count(@initDuration) as coldStarts by bin(1h)

# errors by message
fields @timestamp, @message | filter @message like /ERROR|Traceback|panic/ | sort @timestamp desc | limit 50

# assistant: tool usage and token spend (log usage in the assistant if you want this)
filter @message like /toolCalls/ | parse @message '"inputTokens": *,' as inTok | stats sum(inTok) by bin(1d)
```

### Metrics and alarms (CDK, add to `ApiStack`)

```ts
import * as cw from "aws-cdk-lib/aws-cloudwatch";
import * as cwActions from "aws-cdk-lib/aws-cloudwatch-actions";
import * as sns from "aws-cdk-lib/aws-sns";
import * as subs from "aws-cdk-lib/aws-sns-subscriptions";

const topic = new sns.Topic(this, "Alarms");
if (process.env.ALARM_EMAIL) topic.addSubscription(new subs.EmailSubscription(process.env.ALARM_EMAIL));
const notify = new cwActions.SnsAction(topic);

const alarms: [string, cw.IMetric, number][] = [
  ["Api5xx", this.httpApi.metricServerError({ period: cdk.Duration.minutes(5) }), 5],
  ["ApiFnErrors", apiFn.metricErrors({ period: cdk.Duration.minutes(5) }), 3],
  ["AssistantErrors", assistantFn.metricErrors({ period: cdk.Duration.minutes(5) }), 3],
  ["AssistantThrottles", assistantFn.metricThrottles({ period: cdk.Duration.minutes(5) }), 1],   // hitting the concurrency cap
  ["ApiFnP95", apiFn.metricDuration({ statistic: "p95", period: cdk.Duration.minutes(5) }), 5000],
];
for (const [name, metric, threshold] of alarms) {
  new cw.Alarm(this, name, { metric, threshold, evaluationPeriods: 1,
    comparisonOperator: cw.ComparisonOperator.GREATER_THAN_OR_EQUAL_TO_THRESHOLD,
    treatMissingData: cw.TreatMissingData.NOT_BREACHING }).addAlarmAction(notify);
}
```
Add a DynamoDB `ThrottledRequests`/`SystemErrors` alarm and a CloudFront 5xx alarm (CloudFront metrics live in `us-east-1`).

### Traces and dashboards
`tracing: ACTIVE` sends Lambda segments to **X-Ray**, so you can see assistant → MCP → API as one trace and spot which hop dominates latency. Create a CloudWatch dashboard with: requests, 4xx/5xx, p95 duration per function, cold-start count, DynamoDB consumed RCU/WCU, Bedrock invocation count/latency/tokens (Bedrock publishes runtime metrics to CloudWatch). Define SLOs ([../aws/08](../aws/08-cicd-observability-and-cost.md)), e.g. 99% of `GET /expenses` under 500 ms and assistant success rate > 95%.

### Compare the variants (the point of having three)
Already measured locally from the repo's Dockerfiles (arm64 images, `docker images`): **api-gin 51 MB**, api-fastapi 318 MB, mcp-server 262 MB, assistant 353 MB. The size gap is the main reason Go's Lambda cold start is much shorter; measure `@initDuration` after deploying to confirm for your account.

After a load run (`k6 run --vus 20 --duration 2m script.js` against each URL with a token), fill:

| Metric | react-fastapi | angular-fastapi | react-gin |
|---|---|---|---|
| Image size (ECR) | | | |
| Cold start `@initDuration` | | | |
| Warm p50 / p95 `/api/expenses` | | | |
| Lambda cost per 1M requests (GB-s) | | | |
| SPA transfer size / LCP (Lighthouse) | | | |
Expect FastAPI-vs-Gin differences in init time and memory, and React-vs-Angular differences in bundle size. Having your own numbers is far stronger than quoting folklore.

## 3. Cost control

| Item | Pricing driver | Control |
|---|---|---|
| Bedrock | input/output tokens | `maxTokens`, `MAX_TURNS`, smaller model, reserved concurrency 3, API throttle 20 rps, budget alarm |
| Lambda | requests × GB-seconds | ARM64, right-size memory (power tuning), short timeouts |
| API Gateway HTTP API | per million requests | cheap; throttle |
| DynamoDB on-demand | read/write request units | single-table, `Query` not `Scan`, small items, TTL |
| CloudFront | egress + requests | caching static assets, compression |
| CloudWatch Logs | ingestion + storage | two-week retention, log level INFO, no payload logging |
| ECR / CDK assets | storage | lifecycle policy on the bootstrap repo (keep the last N images) |
| NAT Gateway | **not used** (no VPC) | the architecture deliberately avoids it |

Guardrails: AWS Budgets with forecasted-spend alerts, **Cost Anomaly Detection**, and the `variant`/`stage` tags (activate them as **cost allocation tags** in Billing) so Cost Explorer can group spend by variant.

## 4. Security checklist for this deployment
- [ ] No public Lambda Function URLs; invocation only via API (resource policies)
- [ ] JWT authorizer on all routes except `/health`
- [ ] `adminUserPassword` auth flow only in non-prod
- [ ] Buckets private + TLS-only; CloudFront OAC
- [ ] IAM least privilege (check `cdk diff` for `*` actions, or run `cdk-nag`)
- [ ] Assistant has Bedrock only; MCP has no AWS data permissions
- [ ] CORS limited to the dev origin (production same-origin via CloudFront)
- [ ] Optional hardening: AWS WAF on CloudFront (rate-based rule, managed rule groups), response headers policy (CSP, HSTS), Cognito advanced security/MFA, KMS CMKs, GuardDuty/CloudTrail
- [ ] Prompt-injection posture documented ([../mcp/05](../mcp/05-security-and-deployment-on-aws.md))

## 5. Teardown (practice accounts should be cheap to reset)

```bash
# infra/scripts/destroy.sh <variant> [stage]
set -euo pipefail
V=${1:?}; S=${2:-dev}
cd "$(dirname "$0")/.."
npx cdk destroy --all --force -c variant=$V -c stage=$S -c bedrockModelId=unused
```
Notes:
- Dev resources use `DESTROY` + `autoDeleteObjects`, so buckets and tables go away. **Prod keeps data** (RETAIN, deletion protection): delete those by hand deliberately.
- CloudFront distributions take several minutes to disable and delete.
- CDK **assets** (images in the bootstrap ECR repo, files in the bootstrap bucket) remain; add a lifecycle policy or clean periodically.
- Cognito users vanish with the pool. Log groups created by Lambda remain only if retention was set outside CDK; ours are CDK-managed and deleted.
- Verify nothing billable remains: `aws resourcegroupstaggingapi get-resources --tag-filters Key=app,Values=expense-tracker`.

## 6. Runbook quickies

| Alert | First checks |
|---|---|
| Api5xx / ApiFnErrors | Logs Insights error query; recent deploy? roll back by redeploying previous commit |
| AssistantThrottles | traffic spike vs concurrency cap of 3: legitimate? raise cap and budget deliberately |
| Bedrock `ThrottlingException` | quota/TPM limits; backoff, smaller model, request a quota increase |
| High cold starts | provisioned concurrency for the API function only if latency SLO demands it; shrink image; Go variant |
| DynamoDB throttling | hot key? (all keys start with the user, so look for a bot/user hammering) rate-limit that user |
| Billing alert | Cost Explorer grouped by tag `variant` and by service; check Bedrock and CloudWatch Logs first |

## Exercise
Run the smoke test against all three variants, then run `destroy.sh` on one and confirm the others are unaffected. Write a half-page ADR: "Which variant would I ship and why?" using your measured table.

## Interview Q&A
- **How do you know the deployment works?** Automated, authenticated smoke test plus evals after every deploy; synthetic checks on a schedule.
- **Where do you look first for latency?** X-Ray trace to find the slow hop, REPORT lines for init vs duration, DynamoDB metrics, Bedrock latency.
- **How do you keep an AI feature's bill bounded?** Layered caps: tokens, turns, concurrency, gateway throttling, budgets/anomaly detection.
- **How do you tear down safely?** Dev is disposable by design (DESTROY policies); prod data is retained and needs explicit deletion; tags prove nothing is left.
