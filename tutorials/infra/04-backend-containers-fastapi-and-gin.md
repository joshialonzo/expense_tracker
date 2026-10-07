# Infra 04 — Backend Containers: FastAPI and Gin on Lambda (DynamoDB)

## Lambda Web Adapter (LWA) in one minute
A Lambda *extension* (a binary placed at `/opt/extensions/lambda-adapter`) that:
1. starts alongside your process, waits until your server answers its **readiness check**,
2. converts each API Gateway event into a normal HTTP request to `127.0.0.1:$PORT`,
3. converts your HTTP response back to a Lambda response,
4. adds headers such as `x-amzn-request-context` (the API Gateway request context, including **verified JWT claims**) and `x-amzn-lambda-context`.

Outside Lambda the file is just ignored, so `docker run` behaves like production. Configuration is by environment variables (check the LWA README for the current list and the latest release tag; pin a version):

| Variable | Value | Why |
|---|---|---|
| `AWS_LWA_PORT` | `8080` | where your server listens |
| `AWS_LWA_READINESS_CHECK_PATH` | `/health` | must return a non-5xx quickly |
| `AWS_LWA_INVOKE_MODE` | `buffered` (default) | `response_stream` with Function URLs for streaming |

> **Architecture note:** the code below (identity dependency, DynamoDB repository) is shown in compact form. In the real services they are placed as **adapters** per [../fastapi/07](../fastapi/07-clean-architecture.md) and [../gin/07](../gin/07-clean-architecture.md): the repository implements the `ExpenseReader/Writer` ports and returns domain objects; identity is an HTTP-layer concern that yields a plain `user_id` for the use cases; only `container.py` / `main.go` name concrete classes. Start the Python Docker `CMD` with `uvicorn app.main:create_app --factory ...`.

## Contract the infra expects from *any* backend

| Requirement | Detail |
|---|---|
| Listen on `0.0.0.0:8080` | LWA default |
| `GET /health` → 200 | no auth, no dependencies (liveness/readiness) |
| Identity | take `sub` from the `x-amzn-request-context` header: `requestContext.authorizer.jwt.claims.sub`. **Never** from the body or query. The gateway has already validated the token |
| Config via env | `TABLE_NAME`, `RECEIPTS_BUCKET`, optional `DYNAMODB_ENDPOINT` (local dev) |
| Routes | `/expenses`, `/expenses/{id}`, `/reports/summary`, `/receipts/upload-url` ([../fullstack/01](../fullstack/01-rest-api-design.md)) |
| Logging | JSON to stdout (CloudWatch) |
| Stateless | no local state beyond `/tmp` |

Trust note: reading claims from a header is safe **only because the Lambda can be invoked solely through the authorizer-protected API** (no public Function URL, resource policy restricts the invoker). In local dev, a fake middleware injects a test `sub`.

## FastAPI image

```dockerfile
# api-fastapi/Dockerfile
FROM public.ecr.aws/docker/library/python:3.12-slim
COPY --from=public.ecr.aws/awsguru/aws-lambda-adapter:0.9.1 /lambda-adapter /opt/extensions/lambda-adapter   # pin; check latest tag
ENV PORT=8080 AWS_LWA_PORT=8080 AWS_LWA_READINESS_CHECK_PATH=/health PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app ./app
CMD ["python", "-m", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080"]
```
`requirements.txt`: `fastapi`, `uvicorn[standard]`, `boto3`, `pydantic>=2`, `python-ulid`. (Pin versions in your real file.)

Identity dependency (replaces the Cognito/JWKS verification of [../fastapi/04](../fastapi/04-auth-and-dependencies.md), because the gateway already verified the token):

```python
# app/identity.py
import json, os
from fastapi import Header, HTTPException

def current_user_id(x_amzn_request_context: str | None = Header(default=None)) -> str:
    if x_amzn_request_context:
        ctx = json.loads(x_amzn_request_context)
        return ctx["authorizer"]["jwt"]["claims"]["sub"]
    if os.getenv("DEV_USER_ID"):                    # local development only
        return os.environ["DEV_USER_ID"]
    raise HTTPException(401, "unauthenticated")
```

DynamoDB repository (matches the key design in [01](01-architecture-and-repo-layout.md)):

```python
# app/repo.py
import base64, json, os
from decimal import Decimal
import boto3
from boto3.dynamodb.conditions import Key
from ulid import ULID

_ddb = boto3.resource("dynamodb", endpoint_url=os.getenv("DYNAMODB_ENDPOINT"))   # created once per container
_table = _ddb.Table(os.environ["TABLE_NAME"])

def _clean(item: dict) -> dict:
    out = {k: (int(v) if isinstance(v, Decimal) else v) for k, v in item.items()}
    for k in ("PK", "SK", "GSI1PK", "GSI1SK", "ttl"):
        out.pop(k, None)
    return out

def create(user_id: str, data: dict) -> dict:
    eid = str(ULID())
    item = {"PK": f"USER#{user_id}", "SK": f"EXP#{eid}",
            "GSI1PK": f"USER#{user_id}", "GSI1SK": f"DATE#{data['date']}#{eid}",
            "id": eid, **data}
    _table.put_item(Item=item, ConditionExpression="attribute_not_exists(PK)")
    return _clean(item)

def get(user_id: str, eid: str) -> dict | None:
    r = _table.get_item(Key={"PK": f"USER#{user_id}", "SK": f"EXP#{eid}"})
    return _clean(r["Item"]) if "Item" in r else None          # another user's id simply isn't found → 404

def delete(user_id: str, eid: str) -> None:
    _table.delete_item(Key={"PK": f"USER#{user_id}", "SK": f"EXP#{eid}"})

def list_range(user_id: str, start: str, end: str, limit: int, cursor: str | None):
    kwargs = dict(IndexName="GSI1",
                  KeyConditionExpression=Key("GSI1PK").eq(f"USER#{user_id}") & Key("GSI1SK").between(f"DATE#{start}", f"DATE#{end}~"),
                  ScanIndexForward=False, Limit=limit)
    if cursor:
        kwargs["ExclusiveStartKey"] = json.loads(base64.urlsafe_b64decode(cursor))
    r = _table.query(**kwargs)
    nxt = base64.urlsafe_b64encode(json.dumps(r["LastEvaluatedKey"]).encode()).decode() if "LastEvaluatedKey" in r else None
    return [_clean(i) for i in r["Items"]], nxt
```
Notes: `boto3` returns numbers as `Decimal`, so convert (floats are rejected on write; store cents as `int`). The cursor is the opaque `LastEvaluatedKey`. Money stays integer cents. The route handlers/Pydantic schemas are unchanged from [../fastapi/02](../fastapi/02-pydantic-and-validation.md); only the repository swapped from SQLAlchemy to this. The **monthly summary** queries the month range and sums by category in Python.

Receipts presigned URL endpoint ([../aws/02](../aws/02-s3-and-cloudfront.md)) uses `boto3.client("s3").generate_presigned_url("put_object", ...)` with key `receipts/<sub>/<ulid>`; the Lambda role needs `s3:PutObject/GetObject` on that prefix.

## Gin image

```dockerfile
# api-gin/Dockerfile
FROM public.ecr.aws/docker/library/golang:1.23 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build -trimpath -ldflags="-s -w" -o /out/api ./cmd/api

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=public.ecr.aws/awsguru/aws-lambda-adapter:0.9.1 /lambda-adapter /opt/extensions/lambda-adapter
ENV AWS_LWA_PORT=8080 AWS_LWA_READINESS_CHECK_PATH=/health GIN_MODE=release
COPY --from=build /out/api /api
EXPOSE 8080
ENTRYPOINT ["/api"]
```
Build for `arm64` to match the Lambda architecture (`GOARCH=arm64`). Distroless works because LWA is just a static binary.

Identity middleware (replaces the JWKS middleware of [../gin/04](../gin/04-database-and-auth.md)):

```go
// internal/http/identity.go
func Identity() gin.HandlerFunc {
	type ctxHeader struct {
		Authorizer struct {
			JWT struct{ Claims map[string]string `json:"claims"` } `json:"jwt"`
		} `json:"authorizer"`
	}
	return func(c *gin.Context) {
		if raw := c.GetHeader("x-amzn-request-context"); raw != "" {
			var h ctxHeader
			if err := json.Unmarshal([]byte(raw), &h); err == nil && h.Authorizer.JWT.Claims["sub"] != "" {
				c.Set("userID", h.Authorizer.JWT.Claims["sub"])
				c.Next()
				return
			}
		}
		if dev := os.Getenv("DEV_USER_ID"); dev != "" { // local only
			c.Set("userID", dev)
			c.Next()
			return
		}
		c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"status": 401, "detail": "unauthenticated"})
	}
}
```
Register `/health` **outside** the group that uses `Identity()`.

DynamoDB repository with AWS SDK v2 (`go get github.com/aws/aws-sdk-go-v2/{config,service/dynamodb,feature/dynamodb/attributevalue}` and `github.com/oklog/ulid/v2`):

```go
// internal/expenses/repo_ddb.go
type DDBRepo struct {
	db    *dynamodb.Client
	table string
}

func NewDDBRepo(ctx context.Context, table string) (*DDBRepo, error) {
	cfg, err := config.LoadDefaultConfig(ctx)
	if err != nil { return nil, err }
	opts := []func(*dynamodb.Options){}
	if ep := os.Getenv("DYNAMODB_ENDPOINT"); ep != "" {
		opts = append(opts, func(o *dynamodb.Options) { o.BaseEndpoint = aws.String(ep) })
	}
	return &DDBRepo{db: dynamodb.NewFromConfig(cfg, opts...), table: table}, nil
}

type item struct {
	PK, SK, GSI1PK, GSI1SK string
	Expense                      // embedded: id, amountCents, … via `dynamodbav` tags
}

func (r *DDBRepo) Create(ctx context.Context, userID string, in CreateInput) (Expense, error) {
	id := ulid.Make().String()
	e := Expense{ID: id, AmountCents: in.AmountCents, Currency: cmp.Or(in.Currency, "USD"),
		Category: in.Category, Description: in.Description, SpentOn: in.SpentOn, CreatedAt: time.Now().UTC()}
	av, err := attributevalue.MarshalMap(item{
		PK: "USER#" + userID, SK: "EXP#" + id,
		GSI1PK: "USER#" + userID, GSI1SK: "DATE#" + in.SpentOn + "#" + id, Expense: e})
	if err != nil { return Expense{}, err }
	_, err = r.db.PutItem(ctx, &dynamodb.PutItemInput{
		TableName: &r.table, Item: av, ConditionExpression: aws.String("attribute_not_exists(PK)")})
	return e, err
}

func (r *DDBRepo) Get(ctx context.Context, userID, id string) (Expense, error) {
	out, err := r.db.GetItem(ctx, &dynamodb.GetItemInput{TableName: &r.table, Key: map[string]types.AttributeValue{
		"PK": &types.AttributeValueMemberS{Value: "USER#" + userID},
		"SK": &types.AttributeValueMemberS{Value: "EXP#" + id}}})
	if err != nil { return Expense{}, err }
	if out.Item == nil { return Expense{}, ErrNotFound }
	var e Expense
	return e, attributevalue.UnmarshalMap(out.Item, &e)
}

func (r *DDBRepo) List(ctx context.Context, userID, from, to string, limit int32, cursor map[string]types.AttributeValue) ([]Expense, map[string]types.AttributeValue, error) {
	out, err := r.db.Query(ctx, &dynamodb.QueryInput{
		TableName: &r.table, IndexName: aws.String("GSI1"),
		KeyConditionExpression: aws.String("GSI1PK = :pk AND GSI1SK BETWEEN :a AND :b"),
		ExpressionAttributeValues: map[string]types.AttributeValue{
			":pk": &types.AttributeValueMemberS{Value: "USER#" + userID},
			":a":  &types.AttributeValueMemberS{Value: "DATE#" + from},
			":b":  &types.AttributeValueMemberS{Value: "DATE#" + to + "~"}},
		ScanIndexForward: aws.Bool(false), Limit: &limit, ExclusiveStartKey: cursor})
	if err != nil { return nil, nil, err }
	var items []Expense
	if err := attributevalue.UnmarshalListOfMaps(out.Items, &items); err != nil { return nil, nil, err }
	return items, out.LastEvaluatedKey, nil
}
```
Use `dynamodbav:"amountCents"` tags (or `json` tags via `attributevalue.NewEncoder` options) so stored attribute names match the contract. Encode the cursor as base64 JSON of the key. SDK type names are stable but check them against your installed version.

The `Repo` interface from [../gin/02](../gin/02-gin-rest-api.md) is the seam: `main.go` now builds `NewDDBRepo(ctx, os.Getenv("TABLE_NAME"))` instead of the Postgres repo, and **nothing else changes**, which is exactly the point of the interface.

## Test both against DynamoDB Local before deploying

```bash
# FastAPI
cd api-fastapi && TABLE_NAME=expenses-local DYNAMODB_ENDPOINT=http://localhost:8000 DEV_USER_ID=u1 \
  AWS_ACCESS_KEY_ID=x AWS_SECRET_ACCESS_KEY=x AWS_REGION=us-east-1 uvicorn app.main:app --port 8080
# Gin
cd api-gin && TABLE_NAME=expenses-local DYNAMODB_ENDPOINT=http://localhost:8000 DEV_USER_ID=u1 \
  AWS_ACCESS_KEY_ID=x AWS_SECRET_ACCESS_KEY=x AWS_REGION=us-east-1 go run ./cmd/api
# Same contract test suite against either:
npx schemathesis run http://localhost:8080/openapi.json    # or your pytest/k6 suite hitting :8080
```
Run the **same** contract tests against both back-ends; passing both is the proof that variants are interchangeable.

## CDK: the API function (fragment of `lib/api-stack.ts`)

```ts
import * as path from "path";
import * as cdk from "aws-cdk-lib";
import * as lambda from "aws-cdk-lib/aws-lambda";
import * as logs from "aws-cdk-lib/aws-logs";
import { Construct } from "constructs";

const REPO_ROOT = path.join(__dirname, "..", "..");

/** A container Lambda running a plain HTTP server on :8080 through the Lambda Web Adapter. */
function webFunction(scope: Construct, id: string, dir: string, opts: {
  memorySize?: number; timeout?: cdk.Duration; environment?: Record<string, string>;
  reservedConcurrentExecutions?: number;
}): lambda.DockerImageFunction {
  return new lambda.DockerImageFunction(scope, id, {
    code: lambda.DockerImageCode.fromImageAsset(path.join(REPO_ROOT, dir), {
      platform: cdk.aws_ecr_assets.Platform.LINUX_ARM64,
    }),
    architecture: lambda.Architecture.ARM_64,
    memorySize: opts.memorySize ?? 512,
    timeout: opts.timeout ?? cdk.Duration.seconds(15),
    reservedConcurrentExecutions: opts.reservedConcurrentExecutions,
    tracing: lambda.Tracing.ACTIVE,                          // X-Ray
    logGroup: new logs.LogGroup(scope, `${id}Logs`, {        // explicit log group (logRetention is deprecated)
      retention: logs.RetentionDays.TWO_WEEKS,               // default is forever = surprise bill
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    }),
    environment: {
      AWS_LWA_PORT: "8080",
      AWS_LWA_READINESS_CHECK_PATH: "/health",
      ...opts.environment,
    },
  });
}

// inside the ApiStack constructor:
const apiFn = webFunction(this, "ApiFn", props.variant.apiDir, {
  environment: { TABLE_NAME: props.table.tableName, RECEIPTS_BUCKET: props.receipts.bucketName },
});
props.table.grantReadWriteData(apiFn);                       // generates a least-privilege policy on table + indexes
props.receipts.grantReadWrite(apiFn, "receipts/*");          // prefix-scoped
```
`props.variant.apiDir` is the **only** line that distinguishes FastAPI from Gin infrastructure-wise. `grantReadWriteData` includes the GSI ARNs automatically (a frequent "AccessDenied on index" bug when writing IAM by hand).

## Local container smoke test (same image Lambda runs)

```bash
docker build -t expense-api-fastapi api-fastapi
docker run --rm -p 8080:8080 -e TABLE_NAME=expenses-local -e DEV_USER_ID=u1 \
  -e DYNAMODB_ENDPOINT=http://host.docker.internal:8000 -e AWS_REGION=us-east-1 \
  -e AWS_ACCESS_KEY_ID=x -e AWS_SECRET_ACCESS_KEY=x expense-api-fastapi
curl localhost:8080/health
```
To invoke it as Lambda would (event format), use the Lambda Runtime Interface Emulator pattern from the LWA docs.

## Exercise
Build both images, record image size and **cold start** (`INIT_DURATION` in the CloudWatch REPORT line) after deploying ([06](06-api-gateway-cloudfront-and-frontends.md)). Fill in a table: image MB, init ms, p95 warm latency.

## Interview Q&A
- **Why LWA instead of Mangum or the Go Lambda handler?** One packaging model for all services; local parity; no framework-specific adapters. Cost: an extra process hop and a slightly larger image.
- **Why trust a header for identity?** Only API Gateway can invoke the function, after the authorizer ran; defense in depth by restricting the invoke permission and not exposing a Function URL.
- **How do you stop a Lambda container from being invoked directly with a forged header?** The Lambda resource policy allows only the API's ARN as invoker; IAM controls anyone else.
- **GSI vs filter expression for category?** Filter runs after the read (still consumes capacity); add a GSI if the pattern is hot.
