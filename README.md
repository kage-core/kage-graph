# Kage Knowledge Graph

A community-maintained knowledge graph for Claude Code agents. Validated patterns, gotchas, configurations, and architectural decisions across 17 technology domains — served live via GitHub's CDN. No API key, no install, no backend.

Part of the [Kage memory system](https://github.com/kage-core/Kage). Project and personal memory live in your repos. This graph holds knowledge generic enough to be useful to any developer, anywhere.

---

## How It Works

Agents fetch nodes directly from GitHub's raw CDN:

```
catalog.json          ← domains, hot nodes, total counts
domains/{domain}/
  index.json          ← node list: scores, types, summaries
  nodes/{slug}.md     ← full knowledge node
tags/{tag}.json       ← inverted index for multi-domain queries
```

Every file is a plain HTTP GET. GitHub's CDN serves it globally with no auth and no rate limits for reasonable usage. Nothing to deploy, nothing to maintain.

---

## Retrieval Protocol

The `kage-graph` sub-agent navigates the graph in at most **6 HTTP calls**:

```
Query: "stripe webhook signature fails in express"

Step 0  Hot node cache check                        (0 calls)
        catalog already fetched this session?
        keyword overlaps with hot_nodes? → no match, continue

Step 1  Classify the task                           (0 calls)
        "fails" → type: gotcha, severity: hard-error
        "stripe", "express" → domains: payments, api-design
        tags extracted: ["stripe", "webhook", "signature"]

Step 2  Fetch catalog.json                          (1 call)
        payments: 1 node, top_tags includes "stripe", "webhook" → match
        api-design: 1 node, top_tags includes "webhook", "signature" → match

Step 3  Fetch domain indexes in parallel            (2 calls)
        domains/payments/index.json → stripe-webhook-signature-raw-body (score 50)
        domains/api-design/index.json → webhook-idempotency-key (score 50)
        best match: payments/stripe-webhook-signature-raw-body (3/3 tags)

Step 4  Fetch node                                  (1 call)
        domains/payments/nodes/stripe-webhook-signature-raw-body.md
        no requires edges

Step 5  (budget remaining — fetch related)          (1 call)
        domains/api-design/nodes/webhook-idempotency-key.md
        (complements: HMAC verify + idempotency pattern)

Step 6  Output
        [gotcha] Stripe raw body + [pattern] webhook idempotency
        with scope notes and Never Do examples
```

**Summary pre-selection**: gotcha nodes with `score ≥ 80` can be answered from the index `summary` field alone — no full node fetch needed. Saves 1 call for common lookups.

**Edge resolution**: `requires` edges are auto-fetched within budget. `complements` and `alternative` edges are cited only. `supersedes` silently redirects to the current node.

---

## Node Types

Five types. Each enforces a specific structure so agents can scan them in seconds.

| Type | What it captures | Token limit |
|---|---|---|
| `gotcha` | One failure mode: Symptom → Cause → Fix → Never Do → Scope | 200 |
| `pattern` | Multi-step implementation blueprint with working code | 500 |
| `config` | Version-locked configuration that must be exactly right | 300 |
| `decision` | Architectural trade-off: X over Y, why, when to revise | 400 |
| `reference` | Dense lookup table: error codes, API shapes, CLI flags | 400 |

---

## Node Format

```markdown
---
id: "ai-agents/claude-code-hooks-bypass-permissions"
type: "gotcha"
title: "Claude Code hooks must use --permission-mode bypassPermissions for sub-agents"
domain: "ai-agents"
tags: ["claude-code", "hooks", "agents", "permissions"]
stack: ["claude-code@>=1.0"]
severity: "hard-error"
score: 90
uses: 1
votes: {up: 1, down: 0}
added: "2026-04-12"
updated: "2026-04-12"
fresh: true
ttl_days: 730
supersedes: null
superseded_by: null
related:
  - id: "ai-agents/no-session-persistence-background-agents"
    rel: "requires"
---

# Claude Code hooks must use --permission-mode bypassPermissions for sub-agents

## Symptom
Hook fires, claude process launches, then hangs indefinitely.

## Cause
Hooks run without a TTY. Claude's permission system waits for stdin that never arrives.

## Fix
nohup claude --agent kage-distiller --print "$TASK" \
  --permission-mode bypassPermissions \
  --no-session-persistence \
  >> distill.log 2>&1 &

## Never Do
# Missing --permission-mode → hangs forever on first tool use

## Scope
All Claude Code versions ≥ 1.0. Stop, PreToolUse, PostToolUse, SessionStart hooks.
```

---

## Scoring

```
base:               50
+ votes:            ( up / (up + down + 1) ) × 30     max 30
+ uses:             min( log10(uses+1) / log10(1000) × 20, 20 )  max 20
× staleness:        × 0.85 when TTL expires (fresh → false)
  superseded:       score = 0
─────────────────────────────────────────────────────
max possible:       100
```

Top 5 nodes by score appear in `catalog.json` as `hot_nodes` — agents check these first before spending fetch calls on domain routing.

---

## Lifecycle

**Adding a node** — open a PR with a new file at `domains/{domain}/nodes/{slug}.md`. CI checks schema and blocks if the slug already exists on main. On merge, indexes rebuild automatically.

**Updating a node** — edit the existing file, bump `updated`, reset `fresh: true`. Same PR flow. Use this for content fixes or when the underlying technology changed.

**Superseding a node** — when the approach fundamentally changes, create a new node and link:
```yaml
# new node
supersedes: "domain/old-slug"

# old node (edit in same PR)
superseded_by: "domain/new-slug"
```
The agent silently redirects from old to new at retrieval time.

**Staleness** — the nightly job marks nodes `fresh: false` when their TTL expires and applies a 0.85× score penalty. Stale nodes surface a warning in agent output, prompting someone to update them.

---

## Automation

**On every PR:**
- Schema validated: required fields, type-specific fields, `id` matches file path, slug doesn't already exist on `main`, `conflicts_with` edges bidirectional
- CI must be green before a maintainer can merge

**On every merge to `main`:**
- All `domains/*/index.json`, `catalog.json`, and `tags/*.json` rebuilt atomically from node frontmatter
- Live on GitHub CDN within seconds of merge

**Nightly:**
- Scores recomputed from votes + uses in every node file
- Nodes past TTL marked `fresh: false` with 0.85× score penalty
- Domain indexes and `catalog.json` hot_nodes patched with current scores

---

## Edge Types

```yaml
related:
  - id: "domain/slug"
    rel: "requires"       # auto-fetched when this node is fetched
  - id: "domain/slug"
    rel: "complements"    # cited in output, not fetched
  - id: "domain/slug"
    rel: "alternative"    # same problem, different approach — cited with description
  - id: "domain/slug"
    rel: "supersedes"     # this node is current; that one is outdated
  - id: "domain/slug"
    rel: "specializes"    # version-specific variant of a general node
  - id: "domain/slug"
    rel: "conflicts_with" # cannot apply both — must be declared on both sides
```

---

## Domains

Tags on nodes are **free-form** — use whatever terms describe the specific technology or pattern. The tags below are common ones that have accumulated in each domain and are used by the retrieval agent to classify queries. They are not a whitelist.

| Domain | Common tags |
|---|---|
| `auth` | oauth, jwt, login, session, token, sso, saml, supabase-auth, passkey, magic-link, rbac |
| `database` | postgres, mysql, sqlite, prisma, drizzle, migration, orm, redis, mongodb, planetscale, neon |
| `deployment` | docker, vercel, cloudflare, fly, github-actions, nginx, k8s, railway, render, ecs, helm |
| `frontend` | react, nextjs, vue, svelte, tailwind, ssr, hydration, app-router, vite, astro, remix |
| `testing` | jest, vitest, playwright, cypress, mock, e2e, msw, testing-library, snapshot |
| `api-design` | rest, graphql, trpc, webhook, rate-limit, openapi, pagination, versioning, idempotency |
| `ai-agents` | claude, claude-code, hooks, agents, rag, embeddings, llm, tool-use, prompt, context |
| `payments` | stripe, paddle, billing, subscription, webhook, checkout, refund, proration |
| `storage` | s3, r2, gcs, upload, cdn, blob, presigned-url, multipart, imagekit |
| `email` | smtp, sendgrid, resend, postmark, transactional, deliverability, dkim, spf, dmarc |
| `security` | cors, csp, xss, csrf, sqli, rate-limit, owasp, headers, input-validation, secrets |
| `performance` | caching, cdn, bundle-size, web-vitals, lazy-loading, profiling, lighthouse, ttfb |
| `observability` | logging, metrics, tracing, sentry, datadog, opentelemetry, alerting, dashboards |
| `mobile` | react-native, expo, ios, android, push-notifications, deep-links, offline, fastlane |
| `infrastructure` | terraform, pulumi, aws, gcp, azure, iam, networking, secrets-manager, vpc, cdn |
| `tooling` | typescript, eslint, prettier, webpack, vite, esbuild, tsconfig, monorepo, turborepo |
| `data` | etl, kafka, queues, pipelines, analytics, clickhouse, bigquery, dbt, airflow, flink |

---

## Contributing

Run `/kage submit <node-file>` from Claude Code — it prepares the node and opens the PR. Bots handle validation and review automatically. If the node passes, it merges and goes live within minutes.

See [submissions/template.md](submissions/template.md) for type-specific templates and [CONTRIBUTING.md](CONTRIBUTING.md) for what the review bot checks.
