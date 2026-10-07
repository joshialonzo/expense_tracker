# AWS 04 — Containers: ECR, ECS on Fargate, and the ALB

## Why it matters
ECS is named in the JD. Know the object model (cluster → service → task → container), how traffic and deployments work, and when to choose it over Lambda.

## Object model

| Term | Meaning |
|---|---|
| **ECR** | Private Docker registry. |
| **Task definition** | Versioned blueprint: image, CPU/mem, ports, env, secrets, logging, roles. |
| **Task** | A running instance of a task definition (one or more containers). |
| **Service** | Keeps *N* tasks running, replaces unhealthy ones, integrates with ALB, does rolling deployments, autoscaling. |
| **Cluster** | Logical grouping. |
| **Launch type** | **Fargate** (serverless; you don't manage servers) or EC2 (you manage instances; cheaper at steady high scale / GPUs). |
| **Task role** | What *your code* may call (S3, DynamoDB, Bedrock). |
| **Task execution role** | What *ECS agent* needs to pull the image, write logs, fetch secrets. |

Mixing up task role vs execution role is a classic interview trap.

## Containerize the API (FastAPI example)

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim AS base
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app ./app
RUN useradd -m appuser
USER appuser
EXPOSE 8000
HEALTHCHECK CMD python -c "import urllib.request;urllib.request.urlopen('http://localhost:8000/health')"
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Go version is typically a multi-stage build ending in `FROM gcr.io/distroless/static` — a ~15 MB image (see [../gin/05-deploy-and-testing.md](../gin/05-deploy-and-testing.md)).

## Push to ECR

```bash
aws ecr create-repository --repository-name expense-api
aws ecr get-login-password | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
docker build --platform linux/amd64 -t expense-api .
docker tag expense-api:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/expense-api:$(git rev-parse --short HEAD)
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/expense-api:$(git rev-parse --short HEAD)
```
Tip: on Apple Silicon build with `--platform` matching your Fargate CPU architecture (or choose ARM64 in the task definition). Tag with the git SHA, not only `latest`, so deploys are traceable and rollbacks are trivial.

## Task definition (trimmed)

```json
{
  "family": "expense-api",
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/expense-api-task-role",
  "containerDefinitions": [{
    "name": "api",
    "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/expense-api:abc123",
    "portMappings": [{ "containerPort": 8000 }],
    "environment": [{ "name": "APP_ENV", "value": "prod" }],
    "secrets": [{
      "name": "DATABASE_URL",
      "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:expense/db-AbCdEf"
    }],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/expense-api",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "api"
      }
    }
  }]
}
```

## Networking and traffic

```
Internet → ALB (public subnets, 443, ACM cert)
             └─ Target group (type: ip, health check /health)
                   └─ Fargate tasks (private subnets, SG allows 8000 only from ALB SG)
                          └─ RDS (SG allows 5432 only from task SG)
```

- `awsvpc` mode: every task gets its own ENI and security group.
- Tasks in private subnets need NAT or VPC endpoints (ECR, S3, CloudWatch Logs, Secrets Manager) to pull images and log.
- **Health checks**: ALB health check path must return 200 quickly; failing tasks are replaced. Set a sensible `healthCheckGracePeriodSeconds` for slow starters.

## Deployments and scaling
- **Rolling update** (default): `minimumHealthyPercent`/`maximumPercent` control churn. Deployment **circuit breaker** auto-rolls back failed deployments.
- **Blue/green** via CodeDeploy for traffic shifting.
- **Service auto scaling**: target tracking on CPU (e.g. 60%) or ALB `RequestCountPerTarget`; min 2 tasks across AZs for availability.

```bash
aws ecs register-task-definition --cli-input-json file://taskdef.json
aws ecs update-service --cluster expense --service api --task-definition expense-api:7
aws ecs wait services-stable --cluster expense --services api
```

## Lambda vs ECS Fargate vs App Runner

| Concern | Lambda | ECS Fargate |
|---|---|---|
| Traffic shape | Spiky, idle most of the time | Steady / predictable |
| Cost model | Pay per request/ms (cheap when idle) | Pay per vCPU/GB-hour while running |
| Cold starts | Yes | No (long-running) |
| Execution limit | 15 min | None |
| Long-lived connections (WebSockets, DB pools, streaming) | Awkward | Natural |
| Ops | Least | Moderate (images, scaling, networking) |
| Portability | Lower | Docker anywhere |

For this project: Lambda for the low-traffic API; ECS for the MCP server or an LLM streaming endpoint with long connections. Saying "it depends, here are the axes" beats picking a side.

## Exercise
Containerize the API, push to ECR, run it as a 2-task Fargate service behind an ALB, break the health check path on purpose, and watch ECS replace the tasks.

## Interview Q&A
- **Task role vs execution role?** Execution role = ECS agent plumbing (pull image, logs, secrets). Task role = permissions of your application code.
- **How do you do zero-downtime deploys?** Rolling/blue-green with ALB health checks, min healthy 100%, circuit breaker.
- **How do containers get secrets?** `secrets` in task def referencing Secrets Manager/SSM Parameter Store; never bake into the image.
- **Fargate vs EC2?** Fargate removes node management; EC2 gives control, GPUs, cheaper at sustained scale with Savings Plans/Spot.
- **ECS vs EKS?** ECS is simpler, AWS-native; EKS gives Kubernetes API/portability at the cost of complexity.
