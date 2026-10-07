# Infra 02 — Prerequisites, AWS Account Setup and CDK Bootstrap

## Tools

```bash
node --version      # 20+ (22 recommended)
docker --version    # required: CDK builds the Lambda container images locally
aws --version       # v2
python3 --version   # 3.12 (FastAPI, assistant, mcp-server)
go version          # 1.23+ (Gin)
npm i -g aws-cdk    # CDK CLI (cdk --version)
```
Apple Silicon note: the Lambdas use **arm64** (Graviton, ~20% cheaper), which matches your Mac, so Docker builds are native and fast. In GitHub Actions use an arm64 runner or QEMU/buildx (see [07](07-cicd-github-actions.md)).

## AWS account hygiene (do once)

1. Use a **dedicated practice account** (AWS Organizations member) so you can nuke it freely and set a budget.
2. Don't use the root user for work. Enable MFA on root, then create access through **IAM Identity Center (SSO)**:

```bash
aws configure sso --profile expense-dev        # SSO start URL, region, permission set (AdministratorAccess for a practice account)
aws sso login --profile expense-dev
export AWS_PROFILE=expense-dev
export AWS_REGION=us-east-1                    # pick one region and stay in it (Bedrock model availability varies by region)
aws sts get-caller-identity                    # who am I?
```
3. **Budget alert first** (before deploying anything):

```bash
aws budgets create-budget --account-id $(aws sts get-caller-identity --query Account --output text) \
  --budget '{"BudgetName":"practice-20usd","BudgetLimit":{"Amount":"20","Unit":"USD"},"TimeUnit":"MONTHLY","BudgetType":"COST"}' \
  --notifications-with-subscribers '[{"Notification":{"NotificationType":"ACTUAL","ComparisonOperator":"GREATER_THAN","Threshold":80,"ThresholdType":"PERCENTAGE"},"Subscribers":[{"SubscriptionType":"EMAIL","Address":"you@example.com"}]}]'
```

## Bedrock model access (do this before deploying the AI plane)

1. In the console: **Amazon Bedrock → Model access** (or the equivalent in your console version). Request/enable a text model that supports the **Converse API with tool use** (for example one of the Claude models). Some models require accepting terms, and availability differs per region.
2. Find the exact identifier. Many newer models are invoked through a **cross-region inference profile** ID (prefixed like `us.`), not the bare model ID:

```bash
aws bedrock list-foundation-models --query "modelSummaries[?contains(modelId,'claude')].[modelId,modelName]" --output table
aws bedrock list-inference-profiles --query "inferenceProfileSummaries[].[inferenceProfileId,inferenceProfileName]" --output table
```
3. Smoke-test from the CLI before writing any code:

```bash
aws bedrock-runtime converse --model-id "<id-from-above>" \
  --messages '[{"role":"user","content":[{"text":"Say hi in five words"}]}]' \
  --inference-config '{"maxTokens":50}'
```
If this works, the Lambda will too once it has IAM permission. Write the ID down; it becomes `-c bedrockModelId=<id>` in the CDK deploy. **Never hardcode it from memory**: IDs and availability change.

## Bootstrap CDK (once per account + region)

```bash
cd infra
npx cdk bootstrap aws://$(aws sts get-caller-identity --query Account --output text)/$AWS_REGION
```
This creates the `CDKToolkit` stack: an S3 bucket and an **ECR repository** for assets (Docker images for the Lambdas), plus deploy/lookup IAM roles. Without it `cdk deploy` fails with "SSM parameter /cdk-bootstrap/hnb659fds/version not found".

## Project bootstrap

```bash
mkdir infra && cd infra
npx cdk init app --language typescript
# Tip: TypeScript 7 no longer auto-includes @types/* (set "types": ["node"] in tsconfig) and ts-node depends on the
# old compiler API. The repo runs the CDK app with tsx: "app": "npx tsx bin/app.ts" in cdk.json.
npm i aws-cdk-lib constructs
npm i -D @types/node
```
`cdk init` gives `bin/infra.ts`, `lib/infra-stack.ts`, `cdk.json`, `tsconfig.json`. We'll rename and split the stack in [03](03-cdk-project-data-and-auth.md).

CDK commands you'll use constantly:

| Command | What it does |
|---|---|
| `cdk ls -c variant=react-fastapi` | list the stacks |
| `cdk synth -c variant=…` | generate CloudFormation into `cdk.out/` (no AWS changes) |
| `cdk diff -c variant=…` | what *would* change; read it before every deploy |
| `cdk deploy --all -c variant=…` | deploy (respecting dependencies) |
| `cdk destroy --all -c variant=…` | tear down |
| `cdk watch` | fast hotswap dev loop for Lambda code (dev only) |

## Local dev environment (no AWS needed for the app itself)

```bash
# DynamoDB Local for the backends
docker run -d --name ddb -p 8000:8000 amazon/dynamodb-local
aws dynamodb create-table --endpoint-url http://localhost:8000 --table-name expenses-local \
  --attribute-definitions AttributeName=PK,AttributeType=S AttributeName=SK,AttributeType=S AttributeName=GSI1PK,AttributeType=S AttributeName=GSI1SK,AttributeType=S \
  --key-schema AttributeName=PK,KeyType=HASH AttributeName=SK,KeyType=RANGE \
  --global-secondary-indexes 'IndexName=GSI1,KeySchema=[{AttributeName=GSI1PK,KeyType=HASH},{AttributeName=GSI1SK,KeyType=RANGE}],Projection={ProjectionType=ALL}' \
  --billing-mode PAY_PER_REQUEST
```
Backends read `DYNAMODB_ENDPOINT` (optional) to point at it; the container images run locally exactly as in Lambda because LWA is just a file in `/opt/extensions` that is ignored outside Lambda.

## Cost and quota sanity check (before deploying three variants)
- Lambda, API Gateway HTTP API, DynamoDB on-demand, CloudFront, S3, Cognito (first 10k MAU free in the Essentials tier at time of writing) are all pay-per-use; at practice volume expect cents.
- ECR image storage is cents; CloudWatch Logs ingestion and retention are the usual surprise (we set retention in [08](08-operations-observability-cost-teardown.md)).
- **Bedrock is the only meaningful variable cost**: tokens × price per model. The assistant caps `maxTokens` and tool turns, and is concurrency-limited.
- Verify current pricing on each service's pricing page; numbers change.

## Checklist
- [ ] `aws sts get-caller-identity` works with the SSO profile
- [ ] Budget alert created
- [ ] Bedrock `converse` CLI call succeeds with your chosen model/profile ID
- [ ] `cdk bootstrap` completed
- [ ] Docker running
