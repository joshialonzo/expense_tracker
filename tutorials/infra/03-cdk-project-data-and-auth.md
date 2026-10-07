# Infra 03 — CDK Project, Variant Config, Data and Auth Stacks

## CDK mental model
- **App** → **Stacks** (a unit of deployment = one CloudFormation stack) → **Constructs** (L1 = raw `CfnXxx`, L2 = sensible-default wrappers, L3 = patterns).
- Code **synthesizes** to CloudFormation; CloudFormation does the deploying. Resource properties can be **tokens** (values only known at deploy time, like a table ARN); CDK turns cross-stack token use into exports/imports automatically.
- Everything below is deterministic given the context flags, so `cdk diff` is safe to review in PRs.

## `lib/config.ts` — the variant matrix

```ts
export type Frontend = "react" | "angular";
export type Backend = "fastapi" | "gin";

export interface Variant {
  id: string;
  frontend: Frontend;
  backend: Backend;
  /** local dev origin of the SPA, for Cognito callbacks and CORS */
  devOrigin: string;
  /** where the built SPA ends up (relative to repo root) */
  webDist: string;
  /** backend source folder (relative to repo root) */
  apiDir: string;
}

export const VARIANTS: Record<string, Variant> = {
  "react-fastapi": {
    id: "react-fastapi", frontend: "react", backend: "fastapi",
    devOrigin: "http://localhost:5173", webDist: "web-react/dist", apiDir: "api-fastapi",
  },
  "angular-fastapi": {
    id: "angular-fastapi", frontend: "angular", backend: "fastapi",
    devOrigin: "http://localhost:4200", webDist: "web-angular/dist/web-angular/browser", apiDir: "api-fastapi",
  },
  "react-gin": {
    id: "react-gin", frontend: "react", backend: "gin",
    devOrigin: "http://localhost:5173", webDist: "web-react/dist", apiDir: "api-gin",
  },
};

export interface CommonProps {
  variant: Variant;
  stage: string;           // dev | prod
  prefix: string;          // expense-<variant>-<stage>
}
```
Adding a fourth variant (say `angular-gin`) is one object here, which is the payoff of keeping variants as data.

## `bin/app.ts` — wiring

```ts
#!/usr/bin/env node
import * as cdk from "aws-cdk-lib";
import { VARIANTS } from "../lib/config";
import { DataStack } from "../lib/data-stack";
import { AuthStack } from "../lib/auth-stack";
import { ApiStack } from "../lib/api-stack";
import { WebStack } from "../lib/web-stack";

const app = new cdk.App();

const variantId = app.node.tryGetContext("variant") as string | undefined;
const variant = variantId ? VARIANTS[variantId] : undefined;
if (!variant) throw new Error(`Pass -c variant=<${Object.keys(VARIANTS).join("|")}>`);

const stage = (app.node.tryGetContext("stage") as string | undefined) ?? "dev";
const webOrigin = app.node.tryGetContext("webOrigin") as string | undefined;           // set in pass 2
const bedrockModelId = app.node.tryGetContext("bedrockModelId") as string | undefined;
if (!bedrockModelId) throw new Error("Pass -c bedrockModelId=<model or inference profile id>");

const env = { account: process.env.CDK_DEFAULT_ACCOUNT, region: process.env.CDK_DEFAULT_REGION };
const prefix = `expense-${variant.id}-${stage}`;
const common = { variant, stage, prefix };

const data = new DataStack(app, `${prefix}-data`, { env, ...common, webOrigin });
const auth = new AuthStack(app, `${prefix}-auth`, { env, ...common, webOrigin });
const api = new ApiStack(app, `${prefix}-api`, {
  env, ...common, bedrockModelId,
  table: data.table, receipts: data.receipts, userPool: auth.userPool, client: auth.client,
});
new WebStack(app, `${prefix}-web`, { env, ...common, httpApi: api.httpApi, userPool: auth.userPool, client: auth.client, domain: auth.domain });

cdk.Tags.of(app).add("app", "expense-tracker");
cdk.Tags.of(app).add("variant", variant.id);
cdk.Tags.of(app).add("stage", stage);                // cost allocation tags: group the bill by variant
```
Make `cdk.json` ignore the heavy folders when watching: `"watch": { "exclude": ["**/node_modules", "../web-*/**", "cdk.out"] }`.

## `lib/data-stack.ts` — DynamoDB + receipts bucket

```ts
import * as cdk from "aws-cdk-lib";
import * as dynamodb from "aws-cdk-lib/aws-dynamodb";
import * as s3 from "aws-cdk-lib/aws-s3";
import { Construct } from "constructs";
import { CommonProps } from "./config";

export interface DataStackProps extends cdk.StackProps, CommonProps {
  webOrigin?: string;
}

export class DataStack extends cdk.Stack {
  readonly table: dynamodb.Table;
  readonly receipts: s3.Bucket;

  constructor(scope: Construct, id: string, props: DataStackProps) {
    super(scope, id, props);
    const prod = props.stage === "prod";
    const removal = prod ? cdk.RemovalPolicy.RETAIN : cdk.RemovalPolicy.DESTROY;

    this.table = new dynamodb.Table(this, "Table", {
      partitionKey: { name: "PK", type: dynamodb.AttributeType.STRING },
      sortKey: { name: "SK", type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      pointInTimeRecoverySpecification: { pointInTimeRecoveryEnabled: prod },
      timeToLiveAttribute: "ttl",                   // idempotency keys expire by themselves
      deletionProtection: prod,
      removalPolicy: removal,
    });
    this.table.addGlobalSecondaryIndex({
      indexName: "GSI1",
      partitionKey: { name: "GSI1PK", type: dynamodb.AttributeType.STRING },
      sortKey: { name: "GSI1SK", type: dynamodb.AttributeType.STRING },
      projectionType: dynamodb.ProjectionType.ALL,
    });

    this.receipts = new s3.Bucket(this, "Receipts", {
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      encryption: s3.BucketEncryption.S3_MANAGED,
      enforceSSL: true,
      versioned: prod,
      removalPolicy: removal,
      autoDeleteObjects: !prod,                     // lets `cdk destroy` empty it (adds a helper Lambda)
      lifecycleRules: [{ abortIncompleteMultipartUploadAfter: cdk.Duration.days(1),
                         transitions: [{ storageClass: s3.StorageClass.INFREQUENT_ACCESS, transitionAfter: cdk.Duration.days(90) }] }],
      cors: [{
        allowedMethods: [s3.HttpMethods.PUT, s3.HttpMethods.GET],
        allowedOrigins: [props.variant.devOrigin, ...(props.webOrigin ? [props.webOrigin] : [])],
        allowedHeaders: ["*"],
        maxAge: 3000,
      }],
    });

    new cdk.CfnOutput(this, "TableName", { value: this.table.tableName });
  }
}
```
Talking points: on-demand billing (no capacity planning), TTL on a `ttl` attribute, **RETAIN + deletion protection in prod** versus **DESTROY in dev** (so teardown is clean), bucket is private with `enforceSSL`, CORS needed for browser `PUT` to presigned URLs ([../aws/02](../aws/02-s3-and-cloudfront.md)).

## `lib/auth-stack.ts` — Cognito

```ts
import * as cdk from "aws-cdk-lib";
import * as cognito from "aws-cdk-lib/aws-cognito";
import { Construct } from "constructs";
import { CommonProps } from "./config";

export interface AuthStackProps extends cdk.StackProps, CommonProps {
  webOrigin?: string;
}

export class AuthStack extends cdk.Stack {
  readonly userPool: cognito.UserPool;
  readonly client: cognito.UserPoolClient;
  readonly domain: cognito.UserPoolDomain;

  constructor(scope: Construct, id: string, props: AuthStackProps) {
    super(scope, id, props);
    const prod = props.stage === "prod";

    this.userPool = new cognito.UserPool(this, "Pool", {
      selfSignUpEnabled: true,
      signInAliases: { email: true },
      autoVerify: { email: true },
      standardAttributes: { email: { required: true, mutable: false } },
      passwordPolicy: { minLength: 10, requireLowercase: true, requireUppercase: true, requireDigits: true, requireSymbols: false },
      accountRecovery: cognito.AccountRecovery.EMAIL_ONLY,
      mfa: cognito.Mfa.OPTIONAL,
      mfaSecondFactor: { sms: false, otp: true },
      removalPolicy: prod ? cdk.RemovalPolicy.RETAIN : cdk.RemovalPolicy.DESTROY,
    });

    new cognito.CfnUserPoolGroup(this, "AdminGroup", { userPoolId: this.userPool.userPoolId, groupName: "admin" });

    const callbacks = [props.variant.devOrigin + "/", ...(props.webOrigin ? [props.webOrigin + "/"] : [])];

    this.client = this.userPool.addClient("Web", {
      generateSecret: false,                                            // public SPA client → PKCE, no secret
      authFlows: { userSrp: true, adminUserPassword: !prod },           // admin flow only for smoke tests outside prod
      oAuth: {
        flows: { authorizationCodeGrant: true },                        // no implicit flow
        scopes: [cognito.OAuthScope.OPENID, cognito.OAuthScope.EMAIL, cognito.OAuthScope.PROFILE],
        callbackUrls: callbacks,
        logoutUrls: callbacks,
      },
      accessTokenValidity: cdk.Duration.minutes(60),
      idTokenValidity: cdk.Duration.minutes(60),
      refreshTokenValidity: cdk.Duration.days(30),
      enableTokenRevocation: true,
      preventUserExistenceErrors: true,
      supportedIdentityProviders: [cognito.UserPoolClientIdentityProvider.COGNITO],
    });

    // Hosted UI domain prefix must be globally unique and lowercase
    this.domain = this.userPool.addDomain("Domain", {
      cognitoDomain: { domainPrefix: `${props.prefix}-${this.account}` },
    });

    new cdk.CfnOutput(this, "UserPoolId", { value: this.userPool.userPoolId });
    new cdk.CfnOutput(this, "ClientId", { value: this.client.userPoolClientId });
    new cdk.CfnOutput(this, "HostedUiDomain", { value: this.domain.baseUrl() });
  }
}
```
Notes:
- `webOrigin` is the **two-pass trick**: the CloudFront domain doesn't exist until the web stack deploys, and the web stack depends (through the API) on this one. The first deploy only allows localhost; the second passes `-c webOrigin=https://dxxxx.cloudfront.net` ([06](06-api-gateway-cloudfront-and-frontends.md)). A custom domain you own (Route 53 + ACM) removes the trick, since the URL is known up front.
- `adminUserPassword: !prod` enables the `ADMIN_USER_PASSWORD_AUTH` flow so the smoke test script can get tokens without a browser. Never enable it in prod.
- Tokens are the same ones the earlier theory covered ([../aws/06](../aws/06-cognito-auth-jwt-oauth.md)): the HTTP API authorizer will validate **issuer** and **client id** on the access token.

## Try it: deploy the first two stacks

```bash
cd infra && npm i
MODEL="<your bedrock model or inference profile id>"
npx cdk ls     -c variant=react-fastapi -c bedrockModelId=$MODEL
npx cdk deploy expense-react-fastapi-dev-data expense-react-fastapi-dev-auth -c variant=react-fastapi -c bedrockModelId=$MODEL
```
Then create a test user to confirm Cognito works:

```bash
POOL=$(aws cloudformation describe-stacks --stack-name expense-react-fastapi-dev-auth --query "Stacks[0].Outputs[?OutputKey=='UserPoolId'].OutputValue" --output text)
aws cognito-idp admin-create-user --user-pool-id $POOL --username me@example.com --message-action SUPPRESS
aws cognito-idp admin-set-user-password --user-pool-id $POOL --username me@example.com --password 'Practice-12345' --permanent
```

## Exercise
1. Run `cdk synth` and open the generated template: find the GSI, the TTL spec and the bucket policy that denies non-TLS.
2. Change `stage` to `prod` and `cdk diff`: what changes (retention, deletion protection, PITR)?
3. Add a `variant` for `angular-gin` and confirm `cdk ls` lists it with no other code changes.

## Interview Q&A
- **L1 vs L2 vs L3 constructs?** Raw CloudFormation / opinionated defaults / multi-resource patterns.
- **How do cross-stack references work?** CDK emits CloudFormation exports/imports; they create a hard dependency (can't delete the exporter while imported).
- **Why split into several stacks?** Blast radius, deploy independence, and the 500-resource limit; trade-off is cross-stack coupling.
- **Why RETAIN in prod and DESTROY in dev?** Protect data vs enable clean teardown.
- **Why does the SPA client have no secret?** A browser can't keep one; PKCE protects the code flow.
