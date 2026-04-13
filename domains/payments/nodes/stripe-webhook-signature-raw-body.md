---
id: "payments/stripe-webhook-signature-raw-body"
type: "gotcha"
title: "Stripe webhook signature verification fails when body is parsed before verification"
domain: "payments"
tags: ["stripe", "webhook", "signature", "express", "body-parser", "raw-body"]
stack: ["stripe@>=10", "express@>=4"]
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
---

# Stripe webhook signature verification fails when body is parsed before verification

## Symptom
```
StripeSignatureVerificationError: No signatures found matching the expected signature for payload.
```
`stripe.webhooks.constructEvent()` throws even though the `Stripe-Signature` header is present and the webhook secret is correct. Works with curl but fails in Express.

## Cause
`stripe.webhooks.constructEvent()` requires the **raw request body bytes** to compute the HMAC. If any body-parser middleware runs first (`express.json()`, `bodyParser.json()`), the body is parsed into a JS object — the raw bytes are gone. The signature computed from the re-serialized object never matches Stripe's signature.

## Fix
Mount the Stripe webhook route **before** any `express.json()` middleware, using `express.raw()`:

```typescript
import Stripe from 'stripe'
import express from 'express'

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY)
const app = express()

// Webhook route must come BEFORE express.json()
app.post(
  '/webhooks/stripe',
  express.raw({ type: 'application/json' }),  // raw bytes only
  (req, res) => {
    const sig = req.headers['stripe-signature']

    let event: Stripe.Event
    try {
      event = stripe.webhooks.constructEvent(
        req.body,                              // Buffer, not object
        sig,
        process.env.STRIPE_WEBHOOK_SECRET
      )
    } catch (err) {
      return res.status(400).send(`Webhook error: ${err.message}`)
    }

    switch (event.type) {
      case 'payment_intent.succeeded':
        await fulfillOrder(event.data.object as Stripe.PaymentIntent)
        break
      // handle other events
    }

    res.json({ received: true })
  }
)

// All other routes can use JSON parsing
app.use(express.json())
app.use('/api', apiRouter)
```

## Never Do
```typescript
app.use(express.json())  // ← parses body before webhook route

// This always throws StripeSignatureVerificationError
app.post('/webhooks/stripe', (req, res) => {
  const event = stripe.webhooks.constructEvent(
    req.body,   // already a JS object — signature check fails
    req.headers['stripe-signature'],
    process.env.STRIPE_WEBHOOK_SECRET
  )
})
```

## Scope
Affects Express and any framework where global body-parsing middleware runs before the webhook route. Same issue with Fastify (`fastify.addContentTypeParser`), Hono, and Next.js API routes (which auto-parse bodies — use `export const config = { api: { bodyParser: false } }` in Pages Router, or `req.text()` → `Buffer.from()` in App Router route handlers).
