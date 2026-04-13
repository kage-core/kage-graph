---
id: "storage/s3-presigned-url-content-type"
type: "gotcha"
title: "S3 presigned URL uploads fail when client Content-Type doesn't match signed Content-Type"
domain: "storage"
tags: ["s3", "presigned-url", "upload", "content-type", "aws", "cors"]
stack: ["@aws-sdk/client-s3@>=3", "node@>=16"]
severity: "hard-error"
score: 50
uses: 0
votes: {up: 0, down: 0}
added: "2026-04-13"
updated: "2026-04-13"
fresh: true
ttl_days: 730
supersedes: null
superseded_by: null
related: []
summary: "Presigned URL HMAC includes Content-Type. If client sends a different Content-Type than was signed, S3 returns 403 SignatureDoesNotMatch. Fix: pass the actual file MIME type to the server when generating the URL, then send identical Content-Type header from client."
---

# S3 presigned URL uploads fail when client Content-Type doesn't match signed Content-Type

## Symptom
```
SignatureDoesNotMatch: The request signature we calculated does not match the signature you provided.
```
Presigned PUT URL is generated server-side and returned to the browser. Browser upload fails with 403. Works fine with curl but not `fetch()` or `axios`.

## Cause
The presigned URL signature includes a hash of the request headers. If `Content-Type` is included in the signed headers but the client sends a different `Content-Type` (or omits it, or `fetch` adds a different one), the signature doesn't match. Browsers are inconsistent about what `Content-Type` they attach to `fetch()` PUT requests.

## Fix

**Server — include Content-Type in the presigned URL:**
```typescript
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3'
import { getSignedUrl } from '@aws-sdk/s3-request-presigner'

const s3 = new S3Client({ region: process.env.AWS_REGION })

async function getUploadUrl(key: string, contentType: string) {
  const command = new PutObjectCommand({
    Bucket: process.env.S3_BUCKET,
    Key: key,
    ContentType: contentType,  // lock in the content type
  })

  const url = await getSignedUrl(s3, command, { expiresIn: 300 })
  return { url, contentType }
}
```

**Client — send the exact Content-Type that was signed:**
```typescript
async function uploadFile(file: File) {
  // Get presigned URL — pass actual file MIME type to server
  const { url, contentType } = await fetch('/api/upload-url', {
    method: 'POST',
    body: JSON.stringify({ filename: file.name, contentType: file.type }),
    headers: { 'Content-Type': 'application/json' }
  }).then(r => r.json())

  // Upload directly to S3 — Content-Type must exactly match what was signed
  await fetch(url, {
    method: 'PUT',
    body: file,
    headers: { 'Content-Type': contentType },  // must match signed header
  })
}
```

## Never Do
```typescript
// Server signs with no ContentType — S3 still includes it in sig calculation
const command = new PutObjectCommand({ Bucket, Key })
const url = await getSignedUrl(s3, command)

// Client sends Content-Type anyway — 403 because it wasn't in the signed URL
await fetch(url, { method: 'PUT', body: file, headers: { 'Content-Type': file.type } })
```

## Scope
Affects all AWS SDK versions when using presigned URLs for direct browser uploads. Same behavior with GCS signedURL (`responseType` equivalent) and Cloudflare R2 (S3-compatible). If you don't control the upload client, sign with `ContentType: 'application/octet-stream'` as the fallback and reject unknown types at the application layer.
