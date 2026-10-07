# AWS 02 — S3 and CloudFront: Hosting the React App and Storing Receipts

## Why it matters
S3 is on the JD. Two classic uses: **static website hosting** (the React build) and **object storage for user uploads** (receipt images) via presigned URLs.

## S3 fundamentals
- Buckets are regional, names are globally unique. Objects are keyed (`receipts/user-1/abc.png`); "folders" are just key prefixes.
- Strong read-after-write consistency (since Dec 2020).
- **Block Public Access** ON by default; keep it on. Serve public content through CloudFront instead.
- Storage classes: Standard, Intelligent-Tiering, Standard-IA, Glacier tiers. **Lifecycle rules** transition/expire (e.g. receipts → IA after 90 days).
- Versioning protects against accidental deletes; encryption at rest (SSE-S3 default, SSE-KMS for key control).
- Events: S3 → Lambda/SQS/EventBridge on `ObjectCreated` (e.g. generate thumbnails, OCR receipts).

## Hosting a React SPA: S3 + CloudFront

```
User → CloudFront (HTTPS, caching, custom domain via ACM + Route 53)
          └─ Origin Access Control (OAC) → private S3 bucket
```

Steps:
1. `npm run build` (Vite → `dist/`).
2. Private bucket `expense-tracker-web-dev`, Block Public Access ON.
3. CloudFront distribution, origin = bucket with **OAC**; bucket policy allows only that distribution:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontServicePrincipal",
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::expense-tracker-web-dev/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE"
      }
    }
  }]
}
```

4. **SPA routing**: deep links like `/expenses/42` don't exist as S3 keys. Configure custom error responses: 403 and 404 → `/index.html` with HTTP 200.
5. Default root object `index.html`. Redirect HTTP→HTTPS.
6. Deploy:

```bash
aws s3 sync dist/ s3://expense-tracker-web-dev --delete \
  --cache-control "public,max-age=31536000,immutable" \
  --exclude index.html
aws s3 cp dist/index.html s3://expense-tracker-web-dev/index.html \
  --cache-control "no-cache"
aws cloudfront create-invalidation --distribution-id EDFDVBD6EXAMPLE --paths "/index.html"
```

Caching strategy: hashed asset filenames (`app.3f9a.js`) → cache a year, `index.html` → no-cache. That avoids mass invalidations (which cost money past the free tier).

## Receipts: presigned URL upload

Never stream files through your API (Lambda has a 6 MB sync payload limit). Instead:

```
1. React → POST /receipts/upload-url {contentType}
2. API returns {url, key} (presigned PUT, expires in 5 min)
3. React → PUT file directly to S3 using that URL
4. React → PATCH /expenses/{id} {receiptKey: key}
```

Python (boto3) in the API:

```python
import boto3, uuid
s3 = boto3.client("s3", region_name="us-east-1")

def create_upload_url(user_id: str, content_type: str) -> dict:
    if content_type not in {"image/png", "image/jpeg", "application/pdf"}:
        raise ValueError("unsupported type")
    key = f"receipts/{user_id}/{uuid.uuid4()}"
    url = s3.generate_presigned_url(
        "put_object",
        Params={"Bucket": "expense-tracker-receipts-dev", "Key": key, "ContentType": content_type},
        ExpiresIn=300,
    )
    return {"url": url, "key": key}
```

Browser:

```ts
async function uploadReceipt(file: File): Promise<string> {
  const { url, key } = await api.post<{ url: string; key: string }>("/receipts/upload-url", {
    contentType: file.type,
  });
  const res = await fetch(url, { method: "PUT", body: file, headers: { "Content-Type": file.type } });
  if (!res.ok) throw new Error("upload failed");
  return key;
}
```

Gotchas:
- The `Content-Type` sent must match the one signed.
- Bucket needs **CORS** allowing `PUT` from your site origin.
- Validate ownership: the key prefix must contain the caller's `userId`; reading receipts also uses presigned `GET`.

```json
[{
  "AllowedOrigins": ["https://app.example.com", "http://localhost:5173"],
  "AllowedMethods": ["PUT", "GET"],
  "AllowedHeaders": ["*"],
  "MaxAgeSeconds": 3000
}]
```

## Exercise
Deploy a Vite React build to S3 + CloudFront with OAC, make deep-link refresh work, then add the presigned upload flow end to end.

## Interview Q&A
- **Why CloudFront in front of S3?** HTTPS with custom domains, edge caching/latency, keeps bucket private (OAC), compression, WAF.
- **OAC vs OAI?** OAC is the newer mechanism (supports SSE-KMS, all regions, more methods); OAI is legacy.
- **How do you handle an SPA's client-side routes?** Error-response rewrite to `index.html`, or a CloudFront Function.
- **How do you secure uploads?** Presigned URL with short expiry, content-type and size limits (presigned POST supports a `content-length-range` condition), key namespaced per user, malware scan via S3 event.
- **S3 consistency?** Strong read-after-write for new objects and overwrites.
