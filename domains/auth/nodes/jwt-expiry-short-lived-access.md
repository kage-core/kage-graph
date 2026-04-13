---
id: "auth/jwt-expiry-short-lived-access"
type: "pattern"
title: "Short-lived access tokens + rotating refresh tokens — JWT auth that doesn't require a revocation list"
domain: "auth"
tags: ["jwt", "auth", "refresh-token", "session", "security", "oauth"]
stack: ["node@>=16"]
severity: "warning"
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
summary: "15-min access token (in-memory) + 7-day opaque refresh token (httpOnly cookie). Rotate refresh token on every use — reuse of a rotated token signals theft and invalidates all sessions. No revocation DB needed for normal logouts."
---

# Short-lived access tokens + rotating refresh tokens — JWT auth that doesn't require a revocation list

## Problem
Long-lived JWTs (> 1 hour) can't be revoked on logout without a server-side revocation list — negating the stateless benefit. Short-lived JWTs (< 15 min) require frequent re-login without a refresh mechanism.

## Pattern

### Token structure
```typescript
// Access token — short-lived, stateless
const ACCESS_TTL = 15 * 60 // 15 minutes

// Refresh token — longer-lived, stored in DB, rotated on use
const REFRESH_TTL = 7 * 24 * 60 * 60 // 7 days
```

### Issue tokens on login
```typescript
import jwt from 'jsonwebtoken'
import { randomBytes } from 'crypto'

async function issueTokens(userId: string) {
  const accessToken = jwt.sign(
    { sub: userId, type: 'access' },
    process.env.JWT_SECRET,
    { expiresIn: ACCESS_TTL }
  )

  // Refresh token is opaque — look up in DB, not decoded
  const refreshToken = randomBytes(32).toString('hex')
  const expiresAt = new Date(Date.now() + REFRESH_TTL * 1000)

  await db.refreshToken.create({
    data: { token: refreshToken, userId, expiresAt }
  })

  return { accessToken, refreshToken }
}
```

### Rotate on refresh (detect reuse attacks)
```typescript
async function refreshAccessToken(refreshToken: string) {
  const stored = await db.refreshToken.findUnique({ where: { token: refreshToken } })

  if (!stored || stored.expiresAt < new Date()) {
    // If token not found but family exists, a stolen token was reused — revoke family
    throw new Error('Invalid or expired refresh token')
  }

  // Rotate: delete old, issue new — reuse of old token = theft signal
  await db.refreshToken.delete({ where: { token: refreshToken } })

  return issueTokens(stored.userId)
}
```

### Deliver refresh token via httpOnly cookie
```typescript
res.cookie('refresh_token', refreshToken, {
  httpOnly: true,     // not accessible via JS
  secure: true,       // HTTPS only
  sameSite: 'strict', // no cross-site requests
  maxAge: REFRESH_TTL * 1000,
  path: '/auth/refresh'  // only sent to refresh endpoint
})
res.json({ accessToken })  // access token in memory on client
```

## Key rules
- Access token in-memory on client (not localStorage — XSS risk)
- Refresh token in httpOnly cookie (not accessible to JS)
- Rotate refresh token on every use
- If a refresh token is reused (already deleted), invalidate all sessions for that user (theft detected)
- Never put sensitive claims in JWT payload — they're base64-decodable without the secret

## Scope
Pattern works for any JWT-based auth without a Redis/DB revocation lookup on every request. The tradeoff: revocation during the access token window (up to 15 min) requires either accepting the window or keeping a short TTL blocklist for immediate revocations (admin bans, password resets).
