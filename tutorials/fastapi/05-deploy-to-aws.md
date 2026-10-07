# FastAPI 05 — Deploying to AWS (Lambda and ECS Fargate), Bedrock Endpoint

Pairs with [../aws/03](../aws/03-lambda-and-api-gateway.md) and [../aws/04](../aws/04-ecs-fargate-and-ecr.md).

## Option 1: Lambda + API Gateway with Mangum

```python
# app/lambda_handler.py
from mangum import Mangum
from app.main import app
handler = Mangum(app, lifespan="off")      # API Gateway HTTP API (payload v2) events → ASGI
```

Package as a **container image** (easiest for psycopg/cryptography native wheels):

```dockerfile
FROM public.ecr.aws/lambda/python:3.12
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt --target "${LAMBDA_TASK_ROOT}"
COPY app ${LAMBDA_TASK_ROOT}/app
CMD ["app.lambda_handler.handler"]
```

SAM resource:

```yaml
ApiFn:
  Type: AWS::Serverless::Function
  Properties:
    PackageType: Image
    MemorySize: 1024
    Timeout: 15
    Architectures: [arm64]
    Events:
      Any: { Type: HttpApi, Properties: { ApiId: !Ref Api, Method: ANY, Path: /{proxy+} } }
    Environment:
      Variables: { DATABASE_URL: "{{resolve:secretsmanager:expense/db:SecretString:url}}" }
  Metadata: { DockerContext: ., Dockerfile: Dockerfile.lambda }
```
Lambda tips: small DB pool (`pool_size=1, max_overflow=0`), **RDS Proxy**, Lambda in private subnets with a VPC endpoint or NAT for Secrets Manager/Cognito JWKS, initialize heavy objects at import time (outside handler), keep image lean for cold starts, and `fastapi.middleware` that reads claims from `event["requestContext"]["authorizer"]["jwt"]` is unnecessary since you verify the Bearer token in-app (or trust the gateway authorizer and read claims via `request.scope["aws.event"]`).

## Option 2: ECS Fargate (long-running container)

```dockerfile
FROM python:3.12-slim
ENV PYTHONUNBUFFERED=1 PIP_NO_CACHE_DIR=1
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app ./app
COPY alembic ./alembic
COPY alembic.ini .
RUN useradd -m app && chown -R app /app
USER app
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--proxy-headers", "--forwarded-allow-ips", "*"]
```
- Scale with **tasks**, one uvicorn process per task is fine (the orchestrator handles concurrency); or `gunicorn -k uvicorn.workers.UvicornWorker -w 2` for multi-core tasks.
- `--proxy-headers` so redirects/URLs are right behind the ALB.
- ALB health check: `GET /health` (and a deeper `/ready` that checks the DB).
- **Graceful shutdown**: uvicorn handles SIGTERM; ECS `stopTimeout` ≥ longest request.
- Run migrations via a one-off task: `aws ecs run-task --task-definition expense-migrate ... command: ["alembic","upgrade","head"]` before updating the service.

## Health endpoints

```python
@app.get("/health")                      # liveness: process is up
def health(): return {"status": "ok"}

@app.get("/ready")                       # readiness: dependencies ok
def ready(db: Session = Depends(get_db)):
    db.execute(text("SELECT 1"))
    return {"status": "ready"}
```

> **Clean-architecture note:** the Bedrock call below is shown inline for clarity. In the layered design it becomes a `BedrockInsightGenerator` adapter behind an `InsightGenerator` port used by the `GenerateInsight` use case ([07](07-clean-architecture.md)); the router just calls the use case. Likewise run the app with `uvicorn app.main:create_app --factory`.

## Bedrock-powered endpoint

```python
# app/services/insights.py
import json, boto3
from botocore.config import Config
from app.config import settings
from app.schemas import Insight

_bedrock = boto3.client("bedrock-runtime", region_name=settings.cognito_region,
                        config=Config(read_timeout=60, retries={"max_attempts": 2, "mode": "standard"}))

def generate_insight(totals: list[dict], month: str) -> Insight:
    prompt = (f"Month: {month}\n<totals>{json.dumps(totals)}</totals>\n"
              'Return ONLY JSON: {"summary": str, "anomalies": [str], "tips": [str]}.')
    resp = _bedrock.converse(
        modelId=settings.bedrock_model_id,
        system=[{"text": "You analyze personal spending using only the provided data."}],
        messages=[{"role": "user", "content": [{"text": prompt}]}],
        inferenceConfig={"maxTokens": 600, "temperature": 0.2},
    )
    text = resp["output"]["message"]["content"][0]["text"]
    return Insight.model_validate_json(text)           # fails loudly if the model strays

# router
@router.post("/reports/insights", response_model=Insight)
def insights(month: str, repo: ExpenseRepo = Depends(get_repo)):
    totals = [{"category": c, "totalCents": t, "count": n} for c, t, n in repo.monthly_totals(*month_range(month))]
    if not totals:
        raise HTTPException(404, "no data for that month")
    return generate_insight(totals, month)
```
Boto3 is blocking, so `def` handler (threadpool) is correct here. Add timeouts, catch `ClientError` (`ThrottlingException` → 429/503 with `Retry-After`), per-user rate limits, caching.

IAM: the **task role** / Lambda role needs `bedrock:InvokeModel` for that model ARN ([../aws/07](../aws/07-bedrock-and-ai-integration.md)).

## Streaming to React (SSE)

```python
from fastapi.responses import StreamingResponse

@router.post("/assistant")
def assistant(q: Question, user=Depends(current_user)):
    def gen():
        resp = _bedrock.converse_stream(modelId=settings.bedrock_model_id,
                                        messages=[{"role": "user", "content": [{"text": q.text}]}])
        for event in resp["stream"]:
            delta = event.get("contentBlockDelta", {}).get("delta", {}).get("text")
            if delta:
                yield f"data: {json.dumps({'t': delta})}\n\n"
        yield "data: [DONE]\n\n"
    return StreamingResponse(gen(), media_type="text/event-stream")
```
API Gateway buffers responses; for streaming use ALB→ECS, Lambda response streaming via Function URL, or WebSockets.

## CI/CD
The pipeline in [../aws/08](../aws/08-cicd-observability-and-cost.md): `pytest --cov` → SonarQube → build image → push ECR → (run migration task) → update ECS service (or `sam deploy`).

## Observability
JSON logging (`python-json-logger` or `structlog`), request ID middleware ([01](01-setup-and-routing.md)), AWS Lambda Powertools for Python (logger, tracer, metrics, idempotency utilities) if on Lambda, OpenTelemetry (ADOT collector sidecar) on ECS.

## Exercise
Deploy the same codebase both ways (Lambda via SAM, ECS via task definition), compare cold start, p95 latency under `hey`/`k6` load, and monthly cost at 1M requests. Write the comparison as an ADR in `docs/`.

## Interview Q&A
- **Mangum?** ASGI adapter translating API Gateway/ALB Lambda events to ASGI so FastAPI runs on Lambda.
- **Lambda or ECS for this API?** Traffic profile, cold-start tolerance, connection handling, cost; start Lambda, move if needed.
- **How do you handle migrations on deploy?** One-off task before service update, backward compatible.
- **How do you stop the API from hanging on a slow Bedrock call?** Client timeouts, bounded retries, async/queue for long work, streaming.
