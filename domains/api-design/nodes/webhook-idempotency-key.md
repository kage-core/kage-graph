---
id: "api-design/webhook-idempotency-key"
type: "pattern"
title: "Webhook receivers must verify signatures and process events idempotently"
domain: "api-design"
tags: ["webhook", "idempotency", "signature", "stripe", "hmac", "deduplication"]
stack: ["node@>=16", "express@>=4"]
severity: "hard-error"
score: 50
uses: 0
votes: {up: 0, down: 0}
added: "2026-04-13"
updated: "2026-04-13"
fresh: true
ttl_days: 365
supersedes: null
superseded_by: null
related: []
summary: "Skip HMAC signature verification → spoofable endpoint. Skip idempotency check → double-charges/emails on provider retry. Verify with timingSafeEqual on raw body bytes; deduplicate on delivery ID before any side effects; acknowledge 200 fast and process async."
---

# Webhook receivers must verify signatures and process events idempotently

## Problem
Webhook endpoints that skip signature verification can be spoofed by any attacker who knows the URL. Endpoints that don't track processed events will double-charge customers, send duplicate emails, or over-provision resources when providers retry on network failure.

## Pattern

### 1 — Verify the signature before touching the payload

```typescript
import crypto from 'crypto'

function verifyWebhookSignature(
  rawBody: Buffer,
  signatureHeader: string,
  secret: string
): boolean {
  const expected = crypto
    .createHmac('sha256', secret)
    .update(rawBody)
    .digest('hex')

  // Use timingSafeEqual — avoids timing oracle attacks
  const sig = Buffer.from(signatureHeader.replace('sha256=', ''), 'hex')
  const exp = Buffer.from(expected, 'hex')
  if (sig.length !== exp.length) return false
  return crypto.timingSafeEqual(sig, exp)
}
```

**Critical**: read `req.rawBody`, never `req.body` (parsing strips the original bytes and breaks HMAC).

### 2 — Deduplicate on delivery ID before any side effects

```typescript
// Postgres example — use event_id as unique key
await db.query(`
  INSERT INTO processed_webhooks (event_id, received_at)
  VALUES ($1, NOW())
  ON CONFLICT (event_id) DO NOTHING
  RETURNING event_id
`, [event.id])

if (result.rowCount === 0) {
  // Already processed — return 200 so provider stops retrying
  return res.status(200).json({ status: 'already_processed' })
}

// Side effects only happen here
await fulfillOrder(event.data)
```

### 3 — Return 200 fast, process async

```typescript
app.post('/webhooks', express.raw({ type: 'application/json' }), async (req, res) => {
  if (!verifyWebhookSignature(req.body, req.headers['x-hub-signature-256'], process.env.WEBHOOK_SECRET)) {
    return res.status(401).json({ error: 'Invalid signature' })
  }

  // Acknowledge immediately — providers retry if no 2xx within ~30s
  res.status(200).json({ received: true })

  // Process in background
  processWebhookAsync(req.body).catch(console.error)
})
```

## Never Do
```typescript
// No signature check → anyone can call this
// No deduplication → double-charges on retry
app.post('/webhooks', express.json(), async (req, res) => {
  await chargeCustomer(req.body.customerId, req.body.amount)
  res.json({ ok: true })
})
```

## Scope
Applies to all inbound webhooks: Stripe, GitHub, Linear, Slack, PagerDuty, any provider. Signature header names vary (`X-Hub-Signature-256`, `Stripe-Signature`, `X-Linear-Signature`) — check provider docs. TTL varies per provider (Stripe replays for 72h, GitHub for 3 retries with exponential backoff).
