# Infra 06 — HTTP API, CloudFront and the Two Front-Ends (React and Angular)

This completes the stacks and ends with a working deploy of any variant.

> **Implemented and tested in [`infra/`](../../infra)** (20 CDK assertion tests; `cdk synth` verified for all three variants). Two details beyond the listings: `cdk.json` sets `@aws-cdk/core:defaultCrossStackReferences: strong` (otherwise a warning), and the web stack accepts `-c webDist=<dir>` so `scripts/destroy.sh` can synthesize without a built SPA.

## 1. `lib/api-stack.ts` — final assembled version

```ts
import * as path from "path";
import * as cdk from "aws-cdk-lib";
import * as lambda from "aws-cdk-lib/aws-lambda";
import * as logs from "aws-cdk-lib/aws-logs";
import * as iam from "aws-cdk-lib/aws-iam";
import * as dynamodb from "aws-cdk-lib/aws-dynamodb";
import * as s3 from "aws-cdk-lib/aws-s3";
import * as cognito from "aws-cdk-lib/aws-cognito";
import * as apigw from "aws-cdk-lib/aws-apigatewayv2";
import { HttpLambdaIntegration } from "aws-cdk-lib/aws-apigatewayv2-integrations";
import { HttpJwtAuthorizer } from "aws-cdk-lib/aws-apigatewayv2-authorizers";
import { Construct } from "constructs";
import { CommonProps } from "./config";

const REPO_ROOT = path.join(__dirname, "..", "..");

export interface ApiStackProps extends cdk.StackProps, CommonProps {
  table: dynamodb.ITable;
  receipts: s3.IBucket;
  userPool: cognito.IUserPool;
  client: cognito.IUserPoolClient;
  bedrockModelId: string;
}

export class ApiStack extends cdk.Stack {
  readonly httpApi: apigw.HttpApi;

  constructor(scope: Construct, id: string, props: ApiStackProps) {
    super(scope, id, props);

    // ---- functions (webFunction helper is from tutorial 04) ----
    const apiFn = webFunction(this, "ApiFn", props.variant.apiDir, {
      environment: { TABLE_NAME: props.table.tableName, RECEIPTS_BUCKET: props.receipts.bucketName },
    });
    props.table.grantReadWriteData(apiFn);
    props.receipts.grantReadWrite(apiFn, "receipts/*");

    const mcpFn = webFunction(this, "McpFn", "mcp-server", {});
    const assistantFn = webFunction(this, "AssistantFn", "assistant", {
      timeout: cdk.Duration.seconds(29),
      reservedConcurrentExecutions: 3,
      environment: { BEDROCK_MODEL_ID: props.bedrockModelId, MAX_TURNS: "5" },
    });
    assistantFn.addToRolePolicy(new iam.PolicyStatement({
      actions: ["bedrock:InvokeModel", "bedrock:InvokeModelWithResponseStream"],
      resources: ["arn:aws:bedrock:*::foundation-model/*", `arn:aws:bedrock:${this.region}:${this.account}:inference-profile/*`],
    }));

    // ---- HTTP API + JWT authorizer ----
    const issuer = `https://cognito-idp.${this.region}.amazonaws.com/${props.userPool.userPoolId}`;
    const authorizer = new HttpJwtAuthorizer("CognitoJwt", issuer, {
      jwtAudience: [props.client.userPoolClientId],          // matches `client_id` on Cognito access tokens
      identitySource: ["$request.header.Authorization"],
    });

    this.httpApi = new apigw.HttpApi(this, "Api", {
      apiName: `${props.prefix}-api`,
      defaultAuthorizer: authorizer,                          // everything is authenticated unless opted out
      corsPreflight: {                                        // only matters for local dev against the deployed API
        allowOrigins: [props.variant.devOrigin],
        allowMethods: [apigw.CorsHttpMethod.ANY],
        allowHeaders: ["authorization", "content-type", "idempotency-key", "mcp-session-id", "mcp-protocol-version"],
        maxAge: cdk.Duration.hours(1),
      },
    });

    const apiInt = new HttpLambdaIntegration("ApiInt", apiFn);
    const any = [apigw.HttpMethod.ANY];
    for (const p of ["/expenses", "/expenses/{proxy+}", "/reports/{proxy+}", "/receipts/{proxy+}"]) {
      this.httpApi.addRoutes({ path: p, methods: any, integration: apiInt });
    }
    this.httpApi.addRoutes({                                  // public health check, no auth
      path: "/health", methods: [apigw.HttpMethod.GET], integration: apiInt,
      authorizer: new apigw.HttpNoneAuthorizer(),
    });
    this.httpApi.addRoutes({ path: "/assistant", methods: [apigw.HttpMethod.POST],
      integration: new HttpLambdaIntegration("AssistantInt", assistantFn) });
    this.httpApi.addRoutes({ path: "/mcp", methods: [apigw.HttpMethod.POST, apigw.HttpMethod.GET, apigw.HttpMethod.DELETE],
      integration: new HttpLambdaIntegration("McpInt", mcpFn) });

    // Rate limit the whole stage (protects Bedrock spend and the table)
    (this.httpApi.defaultStage!.node.defaultChild as apigw.CfnStage).defaultRouteSettings = {
      throttlingRateLimit: 20, throttlingBurstLimit: 40,
    };

    new cdk.CfnOutput(this, "ApiUrl", { value: this.httpApi.apiEndpoint });
  }
}
// webFunction(...) defined at the bottom of this file (see tutorial 04)
```
Why it's built this way:
- **`defaultAuthorizer`** flips the safe default: new routes are protected unless you explicitly opt out (only `/health` does).
- `HttpLambdaIntegration` adds the Lambda **resource policy** limiting invocation to this API, which is what makes the header-based identity in [04](04-backend-containers-fastapi-and-gin.md) trustworthy.
- The authorizer is the same for FastAPI and Gin: **auth is infrastructure, not application code**, and one reason the variants are interchangeable.
- Payload format 2.0 (default for HTTP APIs with `HttpLambdaIntegration`) is what LWA expects.

## 2. `lib/web-stack.ts` — S3 + CloudFront + runtime config

```ts
import * as path from "path";
import * as cdk from "aws-cdk-lib";
import * as s3 from "aws-cdk-lib/aws-s3";
import * as s3deploy from "aws-cdk-lib/aws-s3-deployment";
import * as cloudfront from "aws-cdk-lib/aws-cloudfront";
import * as origins from "aws-cdk-lib/aws-cloudfront-origins";
import * as cognito from "aws-cdk-lib/aws-cognito";
import * as apigw from "aws-cdk-lib/aws-apigatewayv2";
import { Construct } from "constructs";
import { CommonProps } from "./config";

export interface WebStackProps extends cdk.StackProps, CommonProps {
  httpApi: apigw.IHttpApi;
  userPool: cognito.IUserPool;
  client: cognito.IUserPoolClient;
  domain: cognito.UserPoolDomain;
}

const SPA_REWRITE = `
function handler(event) {
  var r = event.request;
  // paths without a file extension are client-side routes → serve the SPA shell
  if (r.uri.indexOf('.') === -1) { r.uri = '/index.html'; }
  return r;
}`;
const STRIP_API = `
function handler(event) {
  var r = event.request;
  r.uri = r.uri.replace(/^\\/api/, '') || '/';
  return r;
}`;

export class WebStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: WebStackProps) {
    super(scope, id, props);
    const prod = props.stage === "prod";

    const bucket = new s3.Bucket(this, "SiteBucket", {
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      encryption: s3.BucketEncryption.S3_MANAGED,
      enforceSSL: true,
      removalPolicy: prod ? cdk.RemovalPolicy.RETAIN : cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: !prod,
    });

    const spaRewrite = new cloudfront.Function(this, "SpaRewrite", {
      code: cloudfront.FunctionCode.fromInline(SPA_REWRITE), runtime: cloudfront.FunctionRuntime.JS_2_0,
    });
    const stripApi = new cloudfront.Function(this, "StripApi", {
      code: cloudfront.FunctionCode.fromInline(STRIP_API), runtime: cloudfront.FunctionRuntime.JS_2_0,
    });

    const apiOrigin = new origins.HttpOrigin(`${props.httpApi.apiId}.execute-api.${this.region}.amazonaws.com`, {
      protocolPolicy: cloudfront.OriginProtocolPolicy.HTTPS_ONLY,
      readTimeout: cdk.Duration.seconds(30),
    });

    const distribution = new cloudfront.Distribution(this, "Cdn", {
      defaultRootObject: "index.html",
      // minimumProtocolVersion only takes effect with a custom certificate (add one with a custom domain)
      httpVersion: cloudfront.HttpVersion.HTTP2_AND_3,
      defaultBehavior: {
        origin: origins.S3BucketOrigin.withOriginAccessControl(bucket),     // private bucket via OAC
        viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
        cachePolicy: cloudfront.CachePolicy.CACHING_OPTIMIZED,
        compress: true,
        functionAssociations: [{ function: spaRewrite, eventType: cloudfront.FunctionEventType.VIEWER_REQUEST }],
      },
      additionalBehaviors: {
        "/api/*": {
          origin: apiOrigin,
          viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.HTTPS_ONLY,
          allowedMethods: cloudfront.AllowedMethods.ALLOW_ALL,
          cachePolicy: cloudfront.CachePolicy.CACHING_DISABLED,
          originRequestPolicy: cloudfront.OriginRequestPolicy.ALL_VIEWER_EXCEPT_HOST_HEADER,   // forwards Authorization
          functionAssociations: [{ function: stripApi, eventType: cloudfront.FunctionEventType.VIEWER_REQUEST }],
        },
      },
    });
```
Key details (interview material):
- **No distribution-level custom error responses** for the SPA fallback. They apply to *all* behaviors and would turn API 404s into `index.html`. A viewer-request function limited to the default behavior avoids that bug.
- `ALL_VIEWER_EXCEPT_HOST_HEADER` forwards the `Authorization` header (and the query string) to API Gateway while letting CloudFront set the correct `Host`.
- `CACHING_DISABLED` on `/api/*`: API responses are per-user. Never cache them in a shared cache.
- OAC (not the legacy OAI) keeps the bucket private.

```ts
    // ---- runtime config the SPA fetches at startup (one build works in any environment) ----
    const config = {
      apiBase: "/api",
      region: this.region,
      userPoolId: props.userPool.userPoolId,
      userPoolClientId: props.client.userPoolClientId,
      cognitoDomain: props.domain.baseUrl().replace("https://", ""),
    };

    const dist = path.join(__dirname, "..", "..", props.variant.webDist);

    // immutable, hashed assets: cache for a year
    new s3deploy.BucketDeployment(this, "Assets", {
      sources: [s3deploy.Source.asset(dist)],
      destinationBucket: bucket,
      exclude: ["index.html", "config.json"],
      prune: false,
      cacheControl: [s3deploy.CacheControl.fromString("public,max-age=31536000,immutable")],
    });

    // entry points: never cached; invalidate on every deploy
    new s3deploy.BucketDeployment(this, "Entry", {
      sources: [s3deploy.Source.asset(dist), s3deploy.Source.jsonData("config.json", config)],
      destinationBucket: bucket,
      exclude: ["*"],
      include: ["index.html", "config.json"],
      prune: false,
      cacheControl: [s3deploy.CacheControl.noCache()],
      distribution,
      distributionPaths: ["/index.html", "/config.json"],
    });

    new cdk.CfnOutput(this, "WebUrl", { value: `https://${distribution.distributionDomainName}` });
    new cdk.CfnOutput(this, "ApiViaCdn", { value: `https://${distribution.distributionDomainName}/api` });
  }
}
```
`Source.jsonData` resolves CloudFormation tokens at deploy time (user pool id, domain), so `config.json` is generated from the real stack outputs: no copy/paste of IDs into front-end code. Old hashed assets pile up because `prune:false`; add an S3 lifecycle rule or prune in CI if it matters.

## 3. Front-ends read `/config.json` at startup

### React (`web-react/src/main.tsx`)

```tsx
import { Amplify } from "aws-amplify";
import { createRoot } from "react-dom/client";
import { App } from "./App";

export type RuntimeConfig = { apiBase: string; region: string; userPoolId: string; userPoolClientId: string; cognitoDomain: string };
export let runtimeConfig: RuntimeConfig;

async function start() {
  runtimeConfig = await (await fetch("/config.json", { cache: "no-store" })).json();
  const origin = window.location.origin + "/";
  Amplify.configure({ Auth: { Cognito: {
    userPoolId: runtimeConfig.userPoolId,
    userPoolClientId: runtimeConfig.userPoolClientId,
    loginWith: { oauth: {
      domain: runtimeConfig.cognitoDomain, scopes: ["openid", "email", "profile"],
      redirectSignIn: [origin], redirectSignOut: [origin], responseType: "code" } },
  } } });
  createRoot(document.getElementById("root")!).render(<App />);
}
start();
```
The API client uses `runtimeConfig.apiBase` (`/api` in the cloud) as its base URL; in local dev, Vite proxies `/api` to your local backend (`vite.config.ts`: `server: { proxy: { "/api": { target: "http://localhost:8080", rewrite: p => p.replace(/^\/api/, "") } } }`). For local dev put a `public/config.json` pointing at the deployed Cognito pool.

### Angular (`web-angular/src/main.ts`)

```ts
import { bootstrapApplication } from "@angular/platform-browser";
import { Amplify } from "aws-amplify";
import { appConfig } from "./app/app.config";
import { AppComponent } from "./app/app.component";
import { RUNTIME_CONFIG, RuntimeConfig } from "./app/core/runtime-config";

const cfg: RuntimeConfig = await (await fetch("/config.json", { cache: "no-store" })).json();
const origin = window.location.origin + "/";
Amplify.configure({ Auth: { Cognito: {
  userPoolId: cfg.userPoolId, userPoolClientId: cfg.userPoolClientId,
  loginWith: { oauth: { domain: cfg.cognitoDomain, scopes: ["openid", "email", "profile"],
    redirectSignIn: [origin], redirectSignOut: [origin], responseType: "code" } },
} } });

bootstrapApplication(AppComponent, {
  ...appConfig,
  providers: [...appConfig.providers, { provide: RUNTIME_CONFIG, useValue: cfg }],   // injectable via inject(RUNTIME_CONFIG)
});
```
`RUNTIME_CONFIG` is an `InjectionToken<RuntimeConfig>`; the `ExpenseApi` service ([../angular/02](../angular/02-services-di-http-and-routing.md)) uses `inject(RUNTIME_CONFIG).apiBase` instead of `environment.apiUrl`. Local dev: `proxy.conf.json` (`{"/api": {"target": "http://localhost:8080", "pathRewrite": {"^/api": ""}}}`) with `ng serve --proxy-config proxy.conf.json`.

Both SPAs call the same endpoints (`/api/expenses`, `/api/assistant`, …) with `Authorization: Bearer <access token>`.

## 4. Deploy any variant: `infra/scripts/deploy.sh`

```bash
#!/usr/bin/env bash
# usage: BEDROCK_MODEL_ID=<id> ./scripts/deploy.sh react-fastapi [dev]
set -euo pipefail
VARIANT=${1:?variant (react-fastapi|angular-fastapi|react-gin)}
STAGE=${2:-dev}
: "${BEDROCK_MODEL_ID:?set BEDROCK_MODEL_ID}"
P="expense-${VARIANT}-${STAGE}"
cd "$(dirname "$0")/.."

# 1. build the SPA for this variant (cdk packages the dist folder)
case "$VARIANT" in
  react-*)   (cd ../web-react   && npm ci && npm run build) ;;
  angular-*) (cd ../web-angular && npm ci && npx ng build --configuration production) ;;
esac

CTX=(-c "variant=$VARIANT" -c "stage=$STAGE" -c "bedrockModelId=$BEDROCK_MODEL_ID")

# 2. if the web stack already exists, keep its origin in Cognito callbacks/CORS (otherwise a redeploy would drop it)
ORIGIN=$(aws cloudformation describe-stacks --stack-name "$P-web" \
  --query "Stacks[0].Outputs[?OutputKey=='WebUrl'].OutputValue" --output text 2>/dev/null || true)
[ "$ORIGIN" = "None" ] && ORIGIN=""
[ -n "$ORIGIN" ] && CTX+=(-c "webOrigin=$ORIGIN")

# 3. deploy everything
npx cdk deploy --all --require-approval never "${CTX[@]}"

# 4. first deploy only: now the CloudFront URL exists → add it to Cognito callbacks + receipts CORS
if [ -z "$ORIGIN" ]; then
  ORIGIN=$(aws cloudformation describe-stacks --stack-name "$P-web" \
    --query "Stacks[0].Outputs[?OutputKey=='WebUrl'].OutputValue" --output text)
  npx cdk deploy "$P-data" "$P-auth" --require-approval never "${CTX[@]}" -c "webOrigin=$ORIGIN"
fi
echo "Deployed $P → $ORIGIN"
```

```bash
chmod +x infra/scripts/deploy.sh
cd infra
export BEDROCK_MODEL_ID="<your id>"
./scripts/deploy.sh react-fastapi      # first run takes ~10 minutes (image builds, CloudFront)
./scripts/deploy.sh angular-fastapi
./scripts/deploy.sh react-gin
```
Each creates its own stacks (`expense-react-fastapi-dev-*`, …). They share nothing, so you can compare them or destroy one without touching the others.

## 5. Verify

```bash
URL=$(aws cloudformation describe-stacks --stack-name expense-react-fastapi-dev-web --query "Stacks[0].Outputs[?OutputKey=='WebUrl'].OutputValue" --output text)
curl -s $URL/api/health            # public route through CloudFront → HTTP API → Lambda (first call = cold start)
curl -si $URL/api/expenses | head  # no token → 401 from the authorizer (Lambda never ran)
open $URL                          # sign in via Hosted UI, add an expense, ask the assistant
```
Then run the authenticated smoke test from [08](08-operations-observability-cost-teardown.md).

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Hosted UI: `redirect_mismatch` | callback URL not registered → run pass 2 with `-c webOrigin=…` (the script does) |
| `/api/...` returns the SPA HTML | request matched the default behavior: check the `/api/*` path pattern, or the SPA function applied to it |
| 401 on every call with a valid-looking token | sent the **ID** token (needs the access token), wrong audience (`client_id`), or issuer region/pool mismatch |
| 403 `Forbidden` from CloudFront on `/` | `index.html` not uploaded (SPA build path wrong, e.g. Angular `dist/<name>/browser`) |
| 502/503 from API | container failed readiness: check `/aws/lambda/...` logs; port or `/health` mismatch |
| `AccessDeniedException` on Bedrock | model access not enabled, wrong region, or IAM resource missing the inference profile |
| Timeouts on `/assistant` | tool turns exceeded the 29–30 s limit: reduce `MAX_TURNS` or move to streaming/async |
| `cdk deploy` can't build images | Docker not running, or architecture mismatch (see platform in `webFunction`) |

## Exercise
Deploy the three variants. For each, record: URL, first-request latency (cold), a warm request p50, and the image size from ECR. Then run one assistant question on each and compare the `toolCalls` (they should be identical).

## Interview Q&A
- **Why CloudFront in front of API Gateway?** Single origin for the SPA (no CORS), TLS/HTTP3, WAF attachment point, one domain. Trade-off: extra hop, and cache must be disabled for API responses.
- **How does the SPA know the API/Cognito IDs?** A `config.json` generated by CDK at deploy time and fetched at startup, so one build runs anywhere.
- **Why not use `errorResponses` for SPA routing?** They are distribution-wide and would mask API errors.
- **What authenticates a request at each hop?** Browser→CloudFront: none (public); CloudFront→API GW: JWT; API GW→Lambda: IAM resource policy; Lambda→DynamoDB/Bedrock: execution role.
