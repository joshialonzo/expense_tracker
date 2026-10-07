# AWS 07 — Amazon Bedrock and AI-Powered Features (plus Azure AI Foundry mapping)

## Why it matters
The JD lists "AI-powered features using Azure AI Foundry, AWS Bedrock including pre-built models, prompt engineering, and fine-tuning", plus "CI/CD for AI model deployments". You only run AWS here, so learn Bedrock hands-on and know the Azure equivalents conceptually.

## What Bedrock is
A managed API to many foundation models (Anthropic Claude, Amazon Nova/Titan, Meta Llama, Mistral, Cohere, etc.) with no servers to run. Key points:
- **Model access**: some models need access enabled/subscribed in the account/region. Check the current console/docs; model IDs and **inference profiles** (cross-region IDs like `us.anthropic.…`) change over time. Do not hardcode from memory; put the ID in config.
- Your data is not used to train the base models; calls stay in AWS, can use VPC endpoints (PrivateLink), IAM, KMS, CloudTrail.
- Pricing is per input/output token (on-demand) or provisioned throughput for steady load. Cost control = smaller model, shorter prompts, caching, batch inference.

### Feature map

| Feature | What | Azure AI Foundry analogue |
|---|---|---|
| Model catalog / InvokeModel / **Converse API** | Unified chat API across models | Model catalog + Azure OpenAI / Inference API |
| **Knowledge Bases** | Managed RAG (chunk → embed → vector store → retrieve) | Azure AI Search + "On your data"/Foundry IQ |
| **Agents** | Tool-using orchestration | Foundry Agent Service |
| **Guardrails** | Content/PII/denied-topic filters | Azure AI Content Safety |
| **Prompt management / flows** | Versioned prompts, visual flows | Prompt flow |
| **Model customization** | Fine-tuning, continued pretraining (supported models only), distillation | Fine-tuning in Foundry |
| **Evaluation** | Model/RAG evaluation jobs | Foundry evaluations |

## Calling Bedrock from Python (Converse API)

```python
import boto3, json, os
bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")
MODEL_ID = os.environ["BEDROCK_MODEL_ID"]       # configure; look it up in the console

SYSTEM = [{"text": "You are a concise personal-finance assistant. Use only the data provided."}]

def summarize_spending(expenses: list[dict]) -> str:
    resp = bedrock.converse(
        modelId=MODEL_ID,
        system=SYSTEM,
        messages=[{
            "role": "user",
            "content": [{"text": f"Summarize this month's spending and flag anomalies:\n{json.dumps(expenses)}"}],
        }],
        inferenceConfig={"maxTokens": 500, "temperature": 0.2},
    )
    return resp["output"]["message"]["content"][0]["text"]
```
The Converse API has the same shape for all supported models, so switching models is a config change. Use `converse_stream` for token streaming (pair with an ECS service/Lambda response streaming and SSE to React).

IAM for the task/function role:

```json
{ "Effect": "Allow",
  "Action": ["bedrock:InvokeModel", "bedrock:InvokeModelWithResponseStream"],
  "Resource": "arn:aws:bedrock:us-east-1::foundation-model/<model-id>" }
```
(Inference profiles require permission on the profile ARN too.)

## Prompt engineering (they will ask)
1. **Role + task + constraints** in the system prompt; keep user data in the user turn and clearly delimited (`<expenses>…</expenses>`).
2. **Few-shot examples** to fix output format.
3. **Structured output**: ask for JSON, validate with Pydantic/Zod, retry once on failure. Better: use **tool use** with a JSON schema so the model returns structured arguments.
4. **Chain-of-thought / reason-then-answer** for hard tasks; temperature low for extraction/classification.
5. **Ground it**: give the data (RAG) and instruct "say you don't know if the answer isn't in the context".
6. Treat prompts as code: versioned, reviewed, evaluated against a test set.

### Example: categorize an expense with tool use (structured output)

```python
tool_config = {"tools": [{
  "toolSpec": {
    "name": "set_category",
    "description": "Record the category for an expense",
    "inputSchema": {"json": {
      "type": "object",
      "properties": {
        "category": {"type": "string", "enum": ["food", "transport", "housing", "fun", "other"]},
        "confidence": {"type": "number"}
      },
      "required": ["category"]
    }}
  }
}], "toolChoice": {"tool": {"name": "set_category"}}}

resp = bedrock.converse(
    modelId=MODEL_ID,
    messages=[{"role": "user", "content": [{"text": "Categorize: 'UBER *TRIP 14.20'"}]}],
    toolConfig=tool_config,
)
block = next(b for b in resp["output"]["message"]["content"] if "toolUse" in b)
print(block["toolUse"]["input"])        # {"category": "transport", ...}
```
Tool calling is also how **MCP** works with models (see [../mcp/04-clients-and-bedrock-integration.md](../mcp/04-clients-and-bedrock-integration.md)).

## RAG in one slide
```
Docs → chunk → embed (Titan/Cohere embeddings) → vector store (OpenSearch Serverless / pgvector / Aurora)
Query → embed → top-k similar chunks → stuff into prompt → LLM answers with citations
```
Bedrock Knowledge Bases manages this; `retrieve_and_generate` is the one-call API. pgvector on the RDS you already have is a pragmatic option for a small app.

## Fine-tuning vs prompting vs RAG (classic question)

| Need | Use |
|---|---|
| Different/more *knowledge* that changes | **RAG** |
| Different *behavior/format/tone*, consistent style at scale | Few-shot → then **fine-tuning** |
| Cheaper/faster at same quality on a narrow task | Fine-tune/distill a smaller model |
| Quick start | **Prompting** first, always |

Fine-tuning needs a labeled dataset (JSONL prompt/completion pairs) in S3, a customization job, then **Provisioned Throughput** to serve the custom model. Evaluate before and after.

## Safety, quality, cost
- **Guardrails** for PII redaction/denied topics; never put secrets in prompts; treat model output as untrusted (don't `eval` it, escape it in the UI).
- **Prompt injection**: user/tool data can contain instructions. Isolate, least-privilege tools, human confirmation for destructive actions.
- Log token usage per request; set budgets/alarms; cache repeated prompts; use the smallest adequate model.
- Evaluate with a golden dataset (inputs + expected outputs) in CI.

## CI/CD for AI deployments (JD bullet)
- Prompts, tool schemas and eval datasets live in git.
- PR pipeline: unit tests → **offline eval** of prompt changes against the golden set (fail if score regresses) → deploy to dev.
- Promote via IaC (CDK/SAM/Terraform); model ID, region and guardrail ID are parameters.
- Canary/shadow traffic, monitor latency, error rate, token cost, and quality signals (thumbs up/down).
- For fine-tuned models: pipeline = data validation → customization job → evaluation gate → provisioned throughput → alias switch/rollback.

## Exercise
Add `GET /reports/insights?month=…` that loads the month's expenses, calls Bedrock, validates JSON `{summary, anomalies[], tips[]}` with Pydantic, and returns it. Add a golden-set test with 5 fixtures.

## Interview Q&A
- **Why Converse over InvokeModel?** Uniform request/response and tool-use across models; InvokeModel needs model-specific payloads.
- **How do you reduce hallucinations?** Ground with RAG, constrain output via schema/tools, low temperature, instruct to abstain, verify citations, evals.
- **How do you handle latency?** Streaming, smaller model, shorter context, async job + polling/WebSocket for long tasks.
- **How do you keep costs predictable?** Token budgets, caching, model routing, batch inference, alarms.
- **Azure AI Foundry vs Bedrock?** Same shape (catalog, agents, RAG, guardrails/content safety, evals, fine-tuning); Foundry integrates with Azure OpenAI/Entra; Bedrock with IAM/AWS data stores. Show that you'd keep the app code behind a thin provider interface.
