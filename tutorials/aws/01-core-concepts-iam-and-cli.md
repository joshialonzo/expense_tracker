# AWS 01 — Core Concepts, IAM and the CLI

## Why it matters
Every AWS question eventually becomes "who is allowed to do what, from where, and how do you know". IAM, VPC basics and the shared responsibility model are the vocabulary for the rest of the series.

## Mental model

- **Region** — a geographic area (`us-east-1`). Data and most services are regional. **AZ** — isolated data centers inside a region. Multi-AZ = survive a data center failure; multi-region = survive a region failure (and costs much more).
- **Global services**: IAM, CloudFront, Route 53. Almost everything else is regional.
- **Shared responsibility**: AWS secures *of* the cloud (hardware, hypervisor). You secure *in* the cloud (IAM, data, app, patching what you run, security groups).

## IAM in 5 minutes

| Concept | What it is |
|---|---|
| **User** | Long-lived human/programmatic identity. Avoid for apps; prefer SSO/roles. |
| **Group** | Collection of users sharing policies. |
| **Role** | Identity *assumed* temporarily (by Lambda, ECS task, GitHub Actions, another account). Gives short-lived credentials via STS. |
| **Policy** | JSON document of allow/deny statements. |
| **Identity-based policy** | Attached to user/role: "this identity may do X". |
| **Resource-based policy** | Attached to the resource (S3 bucket, Lambda, SQS): "this principal may do X to me". |
| **Trust policy** | On a role: *who may assume it*. |

Evaluation: **explicit Deny > explicit Allow > implicit Deny** (default). Permissions boundaries and SCPs only *restrict* the maximum.

### Least-privilege policy for the expense API's Lambda

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReceiptsReadWrite",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::expense-tracker-receipts-dev/receipts/*"
    },
    {
      "Sid": "ReadDbSecret",
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:us-east-1:123456789012:secret:expense/db-*"
    }
  ]
}
```

Notice: specific actions, specific ARNs, no `"*"`. Interviewers love hearing "I'd scope `Resource` and use conditions".

Trust policy for Lambda's execution role:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "lambda.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

## Networking essentials (VPC)

- **VPC**: your private network. **Subnets**: public (route to an Internet Gateway) vs private (outbound via NAT Gateway, or none).
- **Security group**: stateful firewall on the ENI/instance. Allow rules only. Reference other SGs (`db-sg` allows 5432 from `api-sg`).
- **NACL**: stateless, subnet-level; rarely the answer.
- Typical layout: ALB in public subnets → ECS tasks in private subnets → RDS in isolated private subnets.
- **VPC endpoints** (gateway for S3/DynamoDB, interface for others) keep traffic off the internet and can avoid NAT cost.
- NAT Gateway is a classic surprise cost (hourly + per-GB).

## CLI practice

```bash
# Use SSO, not long-lived keys
aws configure sso --profile expense-dev
aws sso login --profile expense-dev
export AWS_PROFILE=expense-dev

aws sts get-caller-identity                # "who am I?" — first debugging step
aws s3 ls
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:role/expense-api-role \
  --action-names s3:PutObject \
  --resource-arns arn:aws:s3:::expense-tracker-receipts-dev/receipts/a.png
```

## Reference architecture for this project

```
Browser (React)
  │  HTTPS
  ▼
CloudFront ──► S3 (static SPA)                     [aws/02]
  │
  └─► API Gateway (HTTP API, JWT authorizer)       [aws/03, aws/06]
         ├─► Lambda (Python/Go)  ─┐
         └─► ALB ─► ECS Fargate  ─┼─► RDS Postgres / DynamoDB   [aws/04, aws/05]
                                  └─► Bedrock (insights)        [aws/07]
Cognito (users, tokens)   CloudWatch/X-Ray (observability)      [aws/06, aws/08]
```

## Well-Architected pillars (name them)
Operational excellence, Security, Reliability, Performance efficiency, Cost optimization, Sustainability. When asked to "design X", walk the pillars briefly.

## Exercise
1. Create an SSO profile and run `get-caller-identity`.
2. Create a role with the Lambda trust policy and the least-privilege policy above; use `simulate-principal-policy` to prove `s3:DeleteObject` is denied.
3. Draw the VPC layout above and say which security group rules connect each hop.

## Quick checks
- *Role vs user?* Roles give temporary credentials and have no password/keys; use for workloads.
- *Why can't my Lambda read S3 although the policy is attached?* Check: bucket policy explicit deny, KMS key policy, wrong ARN (`bucket` vs `bucket/*`), permissions boundary/SCP, VPC endpoint policy.
- *Security group vs NACL?* Stateful/instance vs stateless/subnet.
