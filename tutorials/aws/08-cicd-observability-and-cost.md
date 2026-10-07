# AWS 08 — CI/CD, Infrastructure as Code, Observability and Cost

## Why it matters
The JD: "Define and enforce CI/CD pipelines", "Monitor and optimize cloud infrastructure for performance, scalability, and cost", tools: GitHub Actions, SonarQube, GitHub.

## Infrastructure as Code (IaC)
Never click-ops production. Options: **CloudFormation/SAM** (native), **CDK** (CloudFormation via TypeScript/Python, great for TS-heavy teams), **Terraform/OpenTofu** (multi-cloud, state file). Be able to say why: repeatable, reviewable, drift detection, disaster recovery.

A CDK (TypeScript) snippet for a private bucket + CloudFront-ready setup:

```ts
import * as s3 from "aws-cdk-lib/aws-s3";
import { RemovalPolicy, Stack, StackProps } from "aws-cdk-lib";
import { Construct } from "constructs";

export class WebStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);
    new s3.Bucket(this, "WebBucket", {
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      encryption: s3.BucketEncryption.S3_MANAGED,
      versioned: true,
      removalPolicy: RemovalPolicy.RETAIN,
    });
  }
}
```

## CI/CD with GitHub Actions + OIDC (no stored AWS keys)

GitHub gets short-lived credentials by assuming an IAM role through OpenID Connect.

IAM trust policy for the deploy role:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
      "StringLike":   { "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/expense_tracker:ref:refs/heads/master" }
    }
  }]
}
```
Scope `sub` to the repo and branch/environment. This is the answer to "how do you authenticate CI to AWS?"

### Pipeline: test → analyze → build → deploy

```yaml
# .github/workflows/ci-cd.yml
name: ci-cd
on:
  pull_request:
  push:
    branches: [master]

permissions:
  id-token: write      # OIDC
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }          # Sonar needs full history
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm, cache-dependency-path: web/package-lock.json }
      - run: npm ci && npm run lint && npm run typecheck && npm test -- --coverage
        working-directory: web
      - uses: actions/setup-python@v5
        with: { python-version: "3.12", cache: pip }
      - run: pip install -r api/requirements.txt -r api/requirements-dev.txt && pytest --cov=app --cov-report=xml
        working-directory: api
      - name: SonarQube scan
        uses: SonarSource/sonarqube-scan-action@v5
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ vars.SONAR_HOST_URL }}
      - name: Quality gate
        uses: SonarSource/sonarqube-quality-gate-action@v1
        timeout-minutes: 5
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  deploy-api:
    needs: test
    if: github.ref == 'refs/heads/master'
    runs-on: ubuntu-latest
    environment: production            # protection rules / manual approval
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-deploy
          aws-region: us-east-1
      - uses: aws-actions/amazon-ecr-login@v2
        id: ecr
      - name: Build and push
        run: |
          IMAGE=${{ steps.ecr.outputs.registry }}/expense-api:${{ github.sha }}
          docker build -t $IMAGE api && docker push $IMAGE
          echo "IMAGE=$IMAGE" >> $GITHUB_ENV
      - name: Deploy to ECS
        run: |
          aws ecs describe-task-definition --task-definition expense-api --query taskDefinition > td.json
          # render new image into td.json, register, update service (or use aws-actions/amazon-ecs-deploy-task-definition)

  deploy-web:
    needs: test
    if: github.ref == 'refs/heads/master'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci && npm run build
        working-directory: web
      - uses: aws-actions/configure-aws-credentials@v4
        with: { role-to-assume: arn:aws:iam::123456789012:role/gha-deploy, aws-region: us-east-1 }
      - run: |
          aws s3 sync web/dist s3://expense-tracker-web-prod --delete --exclude index.html --cache-control "public,max-age=31536000,immutable"
          aws s3 cp web/dist/index.html s3://expense-tracker-web-prod/index.html --cache-control no-cache
          aws cloudfront create-invalidation --distribution-id ${{ vars.CF_DIST_ID }} --paths /index.html
```
Pin action versions (ideally by SHA for third-party), least-privilege role per environment, required status checks + branch protection, Dependabot, secret scanning, reusable workflows/composite actions for repeated steps.

### SonarQube: what to say
Static analysis for bugs, vulnerabilities, security hotspots, code smells, duplication and coverage. A **quality gate** (e.g. no new bugs/vulnerabilities, coverage on new code ≥ 80%) blocks merges on the *new code* so legacy debt doesn't paralyze you. Complement with `npm audit`/`pip-audit`, Trivy for container images, and SAST like CodeQL.

### Deployment strategies
Rolling (ECS default), blue/green (CodeDeploy for ECS/Lambda aliases, weighted traffic shifting with automatic rollback on CloudWatch alarms), canary. Database changes must be backward compatible (expand/contract) so rollbacks work.

## Observability: the three pillars

| Pillar | AWS | Notes |
|---|---|---|
| Logs | CloudWatch Logs (+ Logs Insights) | Structured JSON, correlation/request ID |
| Metrics | CloudWatch Metrics, **EMF** for custom metrics | Latency p50/p95/p99, error rate, saturation |
| Traces | **X-Ray** / OpenTelemetry (ADOT) | Find the slow hop across gateway → Lambda → DB |

Logs Insights example (top slow routes):

```
fields @timestamp, route, duration_ms
| filter ispresent(duration_ms)
| stats pct(duration_ms, 95) as p95, count() as n by route
| sort p95 desc
| limit 10
```
Alarms (→ SNS → email/Slack/PagerDuty): API 5XX rate, Lambda errors/throttles, ECS CPU/memory + unhealthy host count, RDS CPU/free storage/connections/replica lag, billing. Define **SLIs/SLOs** (e.g. 99.9% of `GET /expenses` < 300 ms) and alert on burn rate, not on every blip. Dashboards for the golden signals: latency, traffic, errors, saturation.

## Cost optimization checklist
- **Visibility first**: cost allocation tags (`app=expense-tracker`, `env=dev`), Cost Explorer, **AWS Budgets** with alerts, Cost Anomaly Detection.
- **Right-size**: Compute Optimizer, Lambda power tuning, Fargate CPU/memory by actual usage; ARM64/Graviton (~20% cheaper).
- **Commitments**: Savings Plans / Reserved for steady load (RDS RIs); Spot for fault-tolerant batch.
- **Eliminate waste**: stop dev environments off-hours, delete orphaned EBS/snapshots/idle load balancers/unattached IPs, S3 lifecycle/Intelligent-Tiering, CloudWatch log retention (default is *forever*), reduce log verbosity.
- **Network**: NAT Gateway data processing and cross-AZ traffic are common surprises; use VPC endpoints (S3/DynamoDB gateway endpoints are free), CloudFront to cut egress.
- **Architecture**: serverless for spiky/idle (pay per use), caching (CloudFront, ElastiCache) to cut DB/compute, DynamoDB on-demand vs provisioned, Aurora Serverless v2 for variable DB load.
- **AI**: model choice, prompt length, caching, batch inference.

## Security checklist (mention briefly)
Least-privilege IAM, MFA/SSO, no long-lived keys, KMS encryption, Secrets Manager, private subnets, WAF on CloudFront/ALB, GuardDuty, CloudTrail, Config, Security Hub, dependency scanning.

## Exercise
Create the OIDC role and a workflow that deploys the SPA to S3/CloudFront on push to `master`. Add a CloudWatch alarm on API 5XX with an SNS email, and a Budget alert at $10.

## Interview Q&A
- **How does GitHub Actions authenticate to AWS?** OIDC → STS `AssumeRoleWithWebIdentity`; trust policy conditions on repo/branch; no static keys.
- **How do you roll back?** Redeploy previous image tag/task definition revision or Lambda alias; blue/green auto-rollback on alarm; DB migrations backward compatible.
- **Walk me through a 3 AM latency alert.** Check dashboard (which route/AZ), X-Ray trace for slow segment, recent deploys, DB metrics (connections, CPU, slow queries), downstream (Bedrock throttling), then mitigate (rollback/scale) before root-causing.
- **Your AWS bill doubled—how do you investigate?** Cost Explorer grouped by service/usage type/tag, look for NAT/data transfer/log ingestion/new resources, anomaly detection, then fix and add budgets.
- **CloudFormation vs Terraform vs CDK?** Native + drift/rollback vs multi-cloud + state vs real languages with abstractions that synthesize to CloudFormation.
