# Kage Knowledge Graph

A community-maintained knowledge graph for Claude Code agents. Validated patterns, gotchas, configurations, and architectural decisions across 10 technology domains — served live via GitHub's CDN. No API key, no install, no backend.

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
Query: "stripe webhook signature fails behind nginx"

Step 0  Hot node cache check                        (0 calls)
        catalog already fetched this session?
        keyword overlaps with hot_nodes? → skip to Step 4

Step 1  Classify the task                           (0 calls)
        "fails" → type: gotcha, severity: hard-error
        "stripe", "nginx" → domains: payments, deployment
        tags extracted: ["stripe", "webhook", "nginx"]

Step 2  Fetch catalog.json                          (1 call)
        payments: 0 nodes → fall through to tag routing

Step 3  Fetch tag files in parallel                 (3 calls)
        tags/stripe.json + tags/webhook.json + tags/nginx.json
        intersection → payments/stripe-webhook-nginx (score 87, 3/3 tags)

Step 4  Fetch node                                  (1 call)
        domains/payments/nodes/stripe-webhook-nginx.md
        has requires edge → deployment/nginx-proxy-buffering

Step 5  Fetch required dependency                   (1 call)
        domains/deployment/nodes/nginx-proxy-buffering.md

Step 6  Output
        [gotcha] Stripe webhook + [config] nginx buffering
        with conflict warnings and related citations
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

The entire contribution pipeline is automated — no human reviewer needed.

**On every PR:**
- Schema validated: required fields, type-specific fields, `id` matches file path, slug doesn't already exist on `main`, `conflicts_with` edges bidirectional
- Claude bot reviews content: specific enough? atomic? scoped to versions? working example present? no PII?
- Both pass → PR auto-merges

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

| Domain | Keywords |
|---|---|
| `auth` | oauth, jwt, login, session, token, sso, saml, supabase-auth |
| `database` | postgres, mysql, sqlite, prisma, drizzle, migration, orm, redis |
| `deployment` | docker, vercel, cloudflare, fly, github-actions, nginx, k8s |
| `frontend` | react, nextjs, vue, svelte, tailwind, ssr, hydration, app-router |
| `testing` | jest, vitest, playwright, cypress, mock, e2e |
| `api-design` | rest, graphql, trpc, webhook, rate-limit, openapi |
| `ai-agents` | claude, claude-code, hooks, agents, rag, embeddings, llm |
| `payments` | stripe, paddle, billing, subscription, webhook |
| `storage` | s3, r2, gcs, upload, cdn, blob |
| `email` | smtp, sendgrid, resend, transactional |

---

## Contributing

Run `/kage submit <node-file>` from Claude Code — it prepares the node and opens the PR. Bots handle validation and review automatically. If the node passes, it merges and goes live within minutes.

See [submissions/template.md](submissions/template.md) for type-specific templates and [CONTRIBUTING.md](CONTRIBUTING.md) for what the review bot checks.
