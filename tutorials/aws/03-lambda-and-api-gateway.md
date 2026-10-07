# AWS 03 — Lambda and API Gateway

## Why it matters
"RESTful APIs on AWS using API Gateway and Lambda" is the headline JD item. Know how a request flows, how to build it, and when *not* to use it.

## Request flow

```
Client → API Gateway (route, authorize, throttle, validate) → Lambda (your code) → RDS/DynamoDB/S3
```

## API Gateway: which flavor?

| | **HTTP API** | **REST API** |
|---|---|---|
| Cost | ~70% cheaper, lower latency | More expensive |
| Auth | JWT authorizer (native), Lambda authorizer, IAM | Cognito authorizer, Lambda authorizer, IAM |
| Extras | Simple, CORS built in | API keys + usage plans, request validation, WAF, caching, private APIs, request/response transforms |

Default to **HTTP API** unless you need REST-only features. WebSocket API exists for realtime.

Integration types: **Lambda proxy** (Lambda gets the full request, returns status/headers/body) is the norm; the payload format v2.0 is for HTTP APIs.

## Lambda essentials
- Event-driven, ephemeral containers. Scale out by concurrency (one request per execution environment at a time).
- **Cold start**: new environment initializes runtime + your init code. Mitigate: smaller packages, init clients outside the handler, avoid VPC-attach pain (now minor), ARM64 (Graviton), provisioned concurrency, SnapStart (Java). Go and Rust cold starts are very small; Python is fine.
- Limits to remember: 15 min max, 10 GB memory (CPU scales with memory), 6 MB sync payload, 250 MB unzipped (10 GB with container image), `/tmp` up to 10 GB.
- **Memory = CPU**: raising memory often makes it cheaper (finishes faster). Use the AWS Lambda Power Tuning tool.
- Idempotency: retries happen (async invokes, SQS). Make handlers idempotent.
- Connections: Lambda can exhaust Postgres connections → use **RDS Proxy**, or DynamoDB.

## A minimal Python handler (payload v2.0)

```python
import json, os
import boto3

dynamodb = boto3.resource("dynamodb")           # created ONCE per environment
table = dynamodb.Table(os.environ["TABLE_NAME"])

def handler(event, context):
    route = event["routeKey"]                    # e.g. "GET /expenses"
    user_id = event["requestContext"]["authorizer"]["jwt"]["claims"]["sub"]

    if route == "GET /expenses":
        res = table.query(
            KeyConditionExpression="PK = :pk AND begins_with(SK, :sk)",
            ExpressionAttributeValues={":pk": f"USER#{user_id}", ":sk": "EXP#"},
            Limit=50,
        )
        return _json(200, res["Items"])

    if route == "POST /expenses":
        body = json.loads(event.get("body") or "{}")
        # validate with pydantic in real code
        item = {"PK": f"USER#{user_id}", "SK": f"EXP#{body['date']}#{body['id']}", **body}
        table.put_item(Item=item)
        return _json(201, item)

    return _json(404, {"message": "not found"})

def _json(status, body):
    return {
        "statusCode": status,
        "headers": {"content-type": "application/json"},
        "body": json.dumps(body, default=str),
    }
```

Key habits: clients created outside the handler, user identity taken from the *authorizer claims* (never from the request body), structured responses.

## FastAPI on Lambda with Mangum
Reuse the same FastAPI app locally and in Lambda (see [../fastapi/05-deploy-to-aws.md](../fastapi/05-deploy-to-aws.md)):

```python
from mangum import Mangum
from app.main import app
handler = Mangum(app)
```

## Infrastructure as code: SAM template

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Runtime: python3.12
    Architectures: [arm64]
    MemorySize: 512
    Timeout: 10
    Environment:
      Variables:
        TABLE_NAME: !Ref ExpensesTable

Resources:
  Api:
    Type: AWS::Serverless::HttpApi
    Properties:
      Auth:
        DefaultAuthorizer: Jwt
        Authorizers:
          Jwt:
            IdentitySource: $request.header.Authorization
            JwtConfiguration:
              issuer: !Sub https://cognito-idp.${AWS::Region}.amazonaws.com/${UserPoolId}
              audience: [!Ref UserPoolClientId]
      CorsConfiguration:
        AllowOrigins: ["https://app.example.com"]
        AllowMethods: [GET, POST, PATCH, DELETE]
        AllowHeaders: [authorization, content-type]

  ExpensesFn:
    Type: AWS::Serverless::Function
    Properties:
      Handler: app.handler
      CodeUri: src/
      Policies:
        - DynamoDBCrudPolicy: { TableName: !Ref ExpensesTable }
      Events:
        List:   { Type: HttpApi, Properties: { ApiId: !Ref Api, Method: GET,  Path: /expenses } }
        Create: { Type: HttpApi, Properties: { ApiId: !Ref Api, Method: POST, Path: /expenses } }

  ExpensesTable:
    Type: AWS::DynamoDB::Table
    Properties:
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - { AttributeName: PK, AttributeType: S }
        - { AttributeName: SK, AttributeType: S }
      KeySchema:
        - { AttributeName: PK, KeyType: HASH }
        - { AttributeName: SK, KeyType: RANGE }

Parameters:
  UserPoolId: { Type: String }
  UserPoolClientId: { Type: String }
```

```bash
sam build && sam deploy --guided
sam local start-api            # local testing
sam logs -n ExpensesFn --tail
```

## Go on Lambda (nice-to-have link)

```go
package main

import (
	"context"
	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
)

func handler(ctx context.Context, req events.APIGatewayV2HTTPRequest) (events.APIGatewayV2HTTPResponse, error) {
	return events.APIGatewayV2HTTPResponse{StatusCode: 200, Body: `{"ok":true}`}, nil
}

func main() { lambda.Start(handler) }
```
Build for `provided.al2023` on `arm64`, binary named `bootstrap`.

## Observability and errors
- Logs go to CloudWatch automatically; log **JSON** with a request ID (`context.aws_request_id`).
- Alarm on `Errors`, `Throttles`, `Duration` p99, API 5XX.
- Async failures → DLQ/on-failure destination. SQS partial batch responses for queue consumers.

## Lambda vs ECS (be ready for this)
See the decision table in [04-ecs-fargate-and-ecr.md](04-ecs-fargate-and-ecr.md).

## Exercise
Deploy the SAM stack, call `POST /expenses` with a Cognito token (see [06](06-cognito-auth-jwt-oauth.md)), then break it: remove the IAM policy and read the CloudWatch error.

## Interview Q&A
- **What causes a 502 from API Gateway + Lambda?** Malformed Lambda response (proxy integration wants `statusCode`, `body` as string), Lambda crash/timeout (504 for gateway timeout at 29 s default limit for REST; HTTP API is 30 s).
- **How do you avoid DB connection exhaustion?** RDS Proxy, reuse connections across invocations, set reserved concurrency, or choose DynamoDB.
- **CORS issue?** Preflight `OPTIONS` must succeed with correct `Access-Control-Allow-*`; HTTP API handles it in config; with credentials you cannot use `*` origin.
- **How do you do auth?** JWT authorizer validates signature/issuer/audience/expiry before invoking Lambda; Lambda reads `claims.sub` for authorization decisions.
- **Reserved vs provisioned concurrency?** Reserved = cap/guarantee for a function; provisioned = pre-warmed environments (cost) to kill cold starts.
