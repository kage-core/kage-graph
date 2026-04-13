---
id: "email/resend-react-email-plain-text"
type: "config"
title: "Resend + React Email requires explicit plain-text fallback to avoid spam filters"
domain: "email"
tags: ["resend", "react-email", "email", "spam", "plain-text", "deliverability"]
stack: ["resend@>=1.0", "@react-email/components@>=0.0.14"]
severity: "warning"
score: 50
uses: 0
votes: {up: 0, down: 0}
added: "2026-04-13"
updated: "2026-04-13"
fresh: true
ttl_days: 180
supersedes: null
superseded_by: null
related: []
---

# Resend + React Email requires explicit plain-text fallback to avoid spam filters

## Symptom
Transactional emails land in spam. Deliverability score tools (mail-tester.com) report missing `text/plain` alternative part. Some corporate mail servers silently discard HTML-only emails.

## Cause
React Email renders HTML by default. Resend sends only the HTML version unless `text` is explicitly provided. Spam filters penalize HTML-only emails because they have no plain-text alternative — a pattern common in bulk/phishing mail.

## Fix

```typescript
import { Resend } from 'resend'
import { render } from '@react-email/components'
import { WelcomeEmail } from './emails/welcome'

const resend = new Resend(process.env.RESEND_API_KEY)

async function sendWelcomeEmail(user: { email: string; name: string }) {
  const emailProps = { name: user.name }

  const html = await render(<WelcomeEmail {...emailProps} />)
  // render with plainText: true strips HTML → produces text fallback
  const text = await render(<WelcomeEmail {...emailProps} />, { plainText: true })

  await resend.emails.send({
    from: 'Acme <hello@acme.com>',
    to: user.email,
    subject: `Welcome to Acme, ${user.name}!`,
    html,
    text,   // required for deliverability
  })
}
```

## Key configuration
```typescript
// Always set both fields on every Resend email.send() call
{
  html: '<rendered HTML string>',
  text: '<plain text fallback>',   // add this
}
```

## Checklist
- [ ] `text` field always provided
- [ ] From address uses a verified domain (not `@resend.dev`)
- [ ] DKIM + SPF configured on the sending domain (Resend dashboard → Domains)
- [ ] Unsubscribe header present for marketing emails: `{ 'List-Unsubscribe': '<mailto:unsub@acme.com>' }`
- [ ] Subject line avoids spam trigger words (test at mail-tester.com)

## Scope
Applies to Resend with React Email. Same requirement exists for Sendgrid (`text` field), Postmark (`TextBody`), and raw SMTP (MIME `text/plain` part). Plain-text content doesn't need to be beautiful — a minimal version of the HTML content is sufficient.
