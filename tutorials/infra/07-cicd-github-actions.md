# Infra 07 — CI/CD with GitHub Actions: One Pipeline, Three Variants

Builds on [../aws/08](../aws/08-cicd-observability-and-cost.md) (OIDC, SonarQube, quality gates). Here the pipeline is **a matrix over the variants**, so every variant is tested and deployed by the same definition.

## Pipeline shape

```
PR:      test (per component) ─► cdk synth (per variant) ─► cdk diff (per variant, read-only role) ─► comment on PR
master:  test ─► deploy dev (matrix: react-fastapi | angular-fastapi | react-gin) ─► smoke test + agent evals
manual:  workflow_dispatch(stage=prod) ─► environment approval ─► deploy prod ─► smoke test
```
Principles: build once per commit, same CDK code for all stages (only context differs), no long-lived AWS keys, approvals only where risk is.

## 1. AWS side: OIDC deploy role (once)

```bash
# one-time per account
aws iam create-open-id-connect-provider --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
```

Trust policy for `gha-expense-deploy` (scope to your repo and branches/environments):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::<ACCOUNT>:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
      "StringLike": { "token.actions.githubusercontent.com:sub": [
        "repo:<OWNER>/<REPO>:ref:refs/heads/master",
        "repo:<OWNER>/<REPO>:environment:dev",
        "repo:<OWNER>/<REPO>:environment:prod" ] }
    }
  }]
}
```
Permissions: the **CDK bootstrap already created narrow roles** (deploy, file-publishing, image-publishing, lookup). The GitHub role only needs to assume them, which is the recommended least-privilege pattern:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": "sts:AssumeRole", "Resource": "arn:aws:iam::<ACCOUNT>:role/cdk-hnb659fds-*" },
    { "Effect": "Allow", "Action": ["cloudformation:DescribeStacks"], "Resource": "arn:aws:cloudformation:*:<ACCOUNT>:stack/expense-*" },
    { "Effect": "Allow", "Action": ["cognito-idp:AdminCreateUser","cognito-idp:AdminSetUserPassword","cognito-idp:AdminInitiateAuth","cognito-idp:AdminDeleteUser"],
      "Resource": "arn:aws:cognito-idp:*:<ACCOUNT>:userpool/*" }
  ]
}
```
(The last statement is for the smoke test user; scope it further by tag or use dev only.) A **separate read-only role for PRs** (`gha-expense-plan`: CloudFormation read + assume the CDK *lookup* role) means a malicious PR can't deploy.

GitHub repo settings: variables `AWS_DEPLOY_ROLE_ARN`, `AWS_PLAN_ROLE_ARN`, `BEDROCK_MODEL_ID`; **environments** `dev` (no approval) and `prod` (required reviewers). Branch protection on `master` requiring the test and synth jobs.

## 2. The workflow

```yaml
# .github/workflows/deploy.yml
name: ci-cd
on:
  push: { branches: [master] }
  pull_request:
  workflow_dispatch:
    inputs:
      stage: { type: choice, options: [dev, prod], default: dev }

permissions:
  id-token: write        # OIDC
  contents: read
  pull-requests: write   # for the diff comment

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}-${{ inputs.stage || 'dev' }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}     # never cancel a running deploy

env:
  AWS_REGION: us-east-1

jobs:
  # ---------- component tests (run once, not per variant) ----------
  web-react:
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: web-react } }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm, cache-dependency-path: web-react/package-lock.json }
      - run: npm ci && npm run lint && npx tsc --noEmit && npm test -- --run && npm run build

  web-angular:
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: web-angular } }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm, cache-dependency-path: web-angular/package-lock.json }
      - run: npm ci && npm run lint --if-present && npx ng test --watch=false --browsers=ChromeHeadless && npx ng build

  api-fastapi:
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: api-fastapi } }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12", cache: pip, cache-dependency-path: api-fastapi/requirements*.txt }
      - run: pip install -r requirements.txt -r requirements-dev.txt && ruff check . && pytest --cov=app --cov-report=xml

  api-gin:
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: api-gin } }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version-file: api-gin/go.mod, cache-dependency-path: api-gin/go.sum }
      - run: go vet ./... && go test -race -coverprofile=coverage.out ./...

  ai-plane:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: |
          for d in assistant mcp-server; do
            pip install -r $d/requirements.txt -r $d/requirements-dev.txt
            (cd $d && ruff check . && pytest -q)
          done

  # ---------- infra: synth + diff for every variant ----------
  infra-plan:
    needs: [web-react, web-angular, api-fastapi, api-gin, ai-plane]
    runs-on: ubuntu-24.04-arm            # native arm64 so Docker builds match the Lambda architecture
    strategy:
      fail-fast: false
      matrix: { variant: [react-fastapi, angular-fastapi, react-gin] }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm, cache-dependency-path: infra/package-lock.json }
      - name: Build SPA for this variant
        run: |
          case "${{ matrix.variant }}" in
            react-*)   (cd web-react   && npm ci && npm run build) ;;
            angular-*) (cd web-angular && npm ci && npx ng build --configuration production) ;;
          esac
      - run: npm ci
        working-directory: infra
      - uses: aws-actions/configure-aws-credentials@v4
        with: { role-to-assume: "${{ vars.AWS_PLAN_ROLE_ARN }}", aws-region: "${{ env.AWS_REGION }}" }
      - name: cdk synth + diff
        working-directory: infra
        run: |
          CTX="-c variant=${{ matrix.variant }} -c stage=dev -c bedrockModelId=${{ vars.BEDROCK_MODEL_ID }}"
          npx cdk synth $CTX --quiet
          npx cdk diff --all $CTX 2>&1 | tee diff.txt || true
      - uses: actions/upload-artifact@v4
        with: { name: "diff-${{ matrix.variant }}", path: infra/diff.txt }

  # ---------- deploy ----------
  deploy:
    if: github.event_name != 'pull_request'
    needs: [infra-plan]
    runs-on: ubuntu-24.04-arm
    environment: ${{ inputs.stage || 'dev' }}
    strategy:
      fail-fast: false
      max-parallel: 3
      matrix: { variant: [react-fastapi, angular-fastapi, react-gin] }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm, cache-dependency-path: infra/package-lock.json }
      - run: npm ci
        working-directory: infra
      - uses: aws-actions/configure-aws-credentials@v4
        with: { role-to-assume: "${{ vars.AWS_DEPLOY_ROLE_ARN }}", aws-region: "${{ env.AWS_REGION }}" }
      - name: Deploy ${{ matrix.variant }}
        working-directory: infra
        env: { BEDROCK_MODEL_ID: "${{ vars.BEDROCK_MODEL_ID }}" }
        run: ./scripts/deploy.sh ${{ matrix.variant }} ${{ inputs.stage || 'dev' }}
      - name: Smoke test (auth, CRUD, MCP, assistant)
        working-directory: infra
        run: ./scripts/smoke-test.sh ${{ matrix.variant }} ${{ inputs.stage || 'dev' }}
```
Points worth explaining:
- **Matrix = proof of interchangeability.** Same tests, same deploy script, same smoke test against three stacks.
- **`ubuntu-24.04-arm`** gives native arm64 builds. On x86 runners add `docker/setup-qemu-action` and `docker/setup-buildx-action` (slower). Check your plan: arm runners may require a paid/public setup.
- **`concurrency` with `cancel-in-progress: false`** for deploys: cancelling a half-finished CloudFormation update is how stacks get stuck in `UPDATE_ROLLBACK_FAILED`.
- **PRs use a read-only role** and only `synth`/`diff`; **merges deploy to `dev`**; **prod is a manual dispatch** behind an approval gate. Alternative: promote on tag.
- **Path filters** (`dorny/paths-filter`) can skip variants whose components didn't change; keep the matrix full while learning.
- Add SonarQube scanning and coverage upload as in [../aws/08](../aws/08-cicd-observability-and-cost.md) and [../fullstack/05](../fullstack/05-testing-and-devops.md); fail on the quality gate.
- Security scans: `npm audit`, `pip-audit`, `govulncheck ./...`, Trivy on the built images, `cdk-nag` (policy-as-code for CDK: flags overly broad IAM, unencrypted buckets, etc.).

## 3. Agent evals as a deploy gate (AI deployment pipeline)

After the smoke test on `dev`, run the eval cases against the deployed assistant ([05](05-ai-plane-mcp-and-bedrock.md)):

```bash
# infra/scripts/run-evals.sh <variant> — reuses the token logic from smoke-test.sh
python evals/run.py --base-url "$URL/api" --token "$TOKEN" --cases evals/cases.json --min-pass 0.9
```
`run.py` posts each question to `/assistant`, then asserts on `toolCalls[].name`, argument shapes, and expected substrings. Fail the job under the threshold. This is the "CI/CD for AI model deployments" bullet from the job description made concrete: **prompt, system text, tool descriptions and model ID are all versioned code, changed by PR, evaluated before they reach `prod`.** Model upgrades are a one-line `BEDROCK_MODEL_ID` variable change that goes through the same gate.

## 4. Release hygiene
- **Drift**: `cdk diff` on a schedule (nightly) to detect console changes.
- **Rollback**: redeploy the previous commit (CDK is declarative; images are tagged by content hash). CloudFormation auto-rolls back failed updates. Data stacks have retention in prod.
- **Order of operations on schema changes**: DynamoDB is schemaless, but item shape changes must be backward compatible (readers tolerate old items) because old and new Lambda versions overlap during deploy.
- **Secrets**: none in the repo or workflow; Cognito/Bedrock need no static credentials. Anything secret later goes in Secrets Manager and is read by the Lambda role.
- **Cost**: three variants × stages multiplies idle resources (still tiny) and build minutes; schedule `destroy` of non-prod ([08](08-operations-observability-cost-teardown.md)).

## Exercise
1. Create the OIDC provider and the two roles; set repo variables; open a PR and read the `diff-*` artifacts.
2. Merge to `master`, watch the three matrix deployments, and open each URL.
3. Introduce a failing smoke assertion (e.g. expect a 200 for an unauthenticated `/api/expenses`) and confirm the pipeline goes red without leaving a broken stack (CloudFormation rolls back).

## Interview Q&A
- **How does CI authenticate to AWS?** OIDC federation → short-lived role; trust policy conditions on repo/branch/environment; the role merely assumes CDK's own narrow deploy roles.
- **How do you stop a PR from a fork or branch from deploying?** Separate read-only role for PRs, deploy only from protected branch/environment, required reviewers on `prod`.
- **How do you roll back a bad deploy?** Redeploy the previous commit; CloudFormation auto-rolls back failed updates; stateful resources are retained.
- **How do you gate AI changes?** Offline evals against a golden set in CI, tool-call assertions, canary in `dev`, monitoring in prod ([08](08-operations-observability-cost-teardown.md)).
- **Why cancel PR runs but not deploy runs?** Stale PR checks are wasted; interrupted CloudFormation updates are dangerous.
