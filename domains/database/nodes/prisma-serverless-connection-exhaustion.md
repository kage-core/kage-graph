---
id: "database/prisma-serverless-connection-exhaustion"
type: "gotcha"
title: "Prisma exhausts Postgres connections in serverless environments"
domain: "database"
tags: ["prisma", "postgres", "serverless", "vercel", "lambda", "connections"]
stack: ["prisma@>=2.0", "postgres@>=12"]
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
summary: "Each serverless function invocation creates a new PrismaClient with its own connection pool. Under load, this exhausts Postgres's connection limit. Fix: singleton PrismaClient + connection_limit=1 in DATABASE_URL, or use PgBouncer/Prisma Accelerate."
---

# Prisma exhausts Postgres connections in serverless environments

## Symptom
App works locally, fails in production on Vercel / Lambda / Cloudflare Workers with:
`Error: too many connections for role` or `remaining connection slots are reserved`.
Error appears under load or after a few minutes of traffic, not on first deploy.

## Cause
Each serverless function invocation instantiates a new `PrismaClient`, which opens its own connection pool. With the default pool size, 10 concurrent invocations open 10 × pool_size connections. Postgres has a hard connection limit (typically 100). The connections are not released between invocations because the function container stays warm.

## Fix
Use a singleton pattern and cap the connection pool to 1 per function instance. Add `connection_limit=1` to the database URL:

```typescript
// lib/prisma.ts
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient }

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    datasources: {
      db: {
        url: process.env.DATABASE_URL + '?connection_limit=1'
      }
    }
  })

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma
```

For high-traffic apps, use PgBouncer or Prisma Accelerate in front of Postgres instead of adjusting `connection_limit`.

## Never Do
```typescript
// Instantiating PrismaClient directly in a handler — new pool per invocation
export async function GET() {
  const prisma = new PrismaClient()
  return prisma.user.findMany()
}
```

## Scope
Affects all serverless runtimes: Vercel Functions, AWS Lambda, Cloudflare Workers, Netlify Functions. Does not affect long-running servers (Next.js on a VPS, Railway, Fly.io) where a single process serves all requests.
