# AWS 09 — Interview Questions and Drills

Answer each **out loud in ~90 seconds**: state the answer, give a reason, name a trade-off.

## Rapid fire
1. Region vs AZ vs edge location?
2. IAM role vs user vs policy? Explicit deny vs allow?
3. Security group vs NACL?
4. S3 consistency model? How do you make a bucket serve a website privately?
5. API Gateway HTTP API vs REST API?
6. Lambda cold start: causes and mitigations?
7. Lambda max timeout / payload / memory? What does memory change?
8. ECS task role vs execution role?
9. Fargate vs EC2 launch type?
10. RDS Multi-AZ vs read replica?
11. DynamoDB: partition key choice, GSI vs LSI, why not Scan?
12. SQS vs SNS vs EventBridge vs Kinesis?
13. Secrets Manager vs Parameter Store?
14. How do you give a container access to a secret?
15. CloudFront invalidation vs versioned file names?
16. What is OIDC federation for GitHub Actions?
17. What is a VPC endpoint? Gateway vs interface?
18. What's in a JWT and how does API Gateway validate it?
19. Bedrock vs running your own model on SageMaker/EC2?
20. How do you stop a prompt injection from deleting data through an agent?

(Answer hints for the messaging question: SQS = queue/pull/decouple; SNS = pub/sub fan-out; EventBridge = event bus with rules/schema and SaaS sources; Kinesis = ordered streaming with replay.)

## Scenario questions

### 1. "Design the expense tracker's backend on AWS."
Cover: SPA on S3+CloudFront; API Gateway HTTP API + Cognito JWT authorizer; Lambda (or ECS) compute; RDS Postgres in private subnets via RDS Proxy; S3 presigned uploads for receipts; Secrets Manager; CloudWatch/X-Ray; IaC + GitHub Actions OIDC; budgets/alarms. Then discuss scale (cache, read replicas), HA (Multi-AZ, ≥2 AZ tasks), DR (backups, cross-region replication, RPO/RTO), security (WAF, KMS, least privilege) and cost.

### 2. "Your Lambda API is slow and sometimes returns 502."
Check CloudWatch logs for timeouts/errors, response format, cold starts (`Init Duration`), DB connections (RDS Proxy), VPC/NAT issues, memory sizing, downstream latency (X-Ray), API Gateway 29/30 s limit.

### 3. "Users upload 50 MB receipts."
Presigned URL/multipart upload directly to S3, validate type/size, S3 event → Lambda for virus scan/thumbnail/OCR (Textract), store key in DB.

### 4. "Traffic spikes 100× during tax season."
Lambda concurrency limits and account quotas, DynamoDB on-demand or RDS Proxy + read replicas + caching, CloudFront caching, queue writes with SQS, autoscaling policies, load test first, pre-warm/provisioned concurrency.

### 5. "Add an AI assistant that answers questions about a user's spending."
Tool-using model on Bedrock with tools = read-only query functions scoped to the caller's `sub`; MCP server exposing those tools ([../mcp/](../mcp/)); guardrails; streaming; per-user rate limits and token budgets; log redacted prompts; evals in CI.

### 6. "Multi-region failover?"
Route 53 health checks + failover/latency routing, DynamoDB Global Tables or Aurora Global Database, S3 CRR, IaC to stamp out a second region, rehearse failover; state the cost vs RPO/RTO trade-off and that most apps only need multi-AZ.

### 7. "Production deploy broke checkout (or: add-expense)."
Roll back first (previous task definition/Lambda alias), then investigate with a diff of the deploy, logs, traces; add a failing test/alarm; write a blameless postmortem.

## Behavioral hooks tied to the JD
Have a 2-minute story ready for each: a cost reduction you drove; a production incident you handled; a CI/CD improvement; a time you disagreed in code review; how you stay current (release notes, re:Invent recaps, side projects like this one).

## Hands-on checklist before the interview
- [ ] Deploy SPA to S3+CloudFront
- [ ] Lambda + HTTP API + JWT authorizer working with Cognito
- [ ] Same API containerized on ECS Fargate behind ALB
- [ ] RDS Postgres reachable only from the app SG
- [ ] Bedrock `converse` call from code
- [ ] GitHub Actions deploying via OIDC
- [ ] CloudWatch alarm + budget set
