# Kage Knowledge Graph

A community-maintained knowledge graph for Claude Code agents. Validated patterns, gotchas, configurations, and architectural decisions across 10 technology domains — served live via GitHub's CDN, no API key required.

This is **Tier 3** of the [Kage memory system](https://github.com/Kage18/Kage). Project and personal memory live in your repos. This graph holds knowledge generic enough to be useful to anyone.

---

## What Lives Here

Five types of nodes, each with a strict format:

| Type | Definition | Example |
|---|---|---|
| `gotcha` | One atomic failure mode: symptom → cause → fix. Max 200 tokens. | "Stripe webhook signature fails behind nginx" |
| `pattern` | Multi-step implementation blueprint with working code | "Supabase SSR session refresh with Next.js App Router" |
| `config` | Version-locked configuration that must be exactly right | "Dockerfile for Node 20 with pnpm on Alpine" |
| `decision` | Architectural trade-off frozen at a point in time | "Drizzle over Prisma for edge runtimes" |
| `reference` | Dense lookup table: error codes, API shapes, CLI flags | "Stripe webhook event types" |

---

## How Agents Use This

The `kage-graph` sub-agent fetches from this repo using raw GitHub CDN URLs:

```
https://raw.githubusercontent.com/Kage18/kage-graph/main/catalog.json
https://raw.githubusercontent.com/Kage18/kage-graph/main/domains/{domain}/index.json
https://raw.githubusercontent.com/Kage18/kage-graph/main/domains/{domain}/nodes/{slug}.md
https://raw.githubusercontent.com/Kage18/kage-graph/main/tags/{tag}.json
```

**Retrieval protocol** (max 6 HTTP calls per query):

1. Check hot node cache (no call) — if `catalog.json` was already fetched and a keyword matches a hot node, jump directly to Step 4
2. Detect task type from symptom language (no call) — maps to gotcha/pattern/config/decision/reference
3. Fetch `catalog.json` — confirm domains have nodes, check hot_nodes
4. Route to nodes via domain index (1 call) or tag intersection (up to 3 parallel calls)
5. Fetch 1–2 node files; auto-follow `requires` edges
6. Return formatted output with conflict warnings and related citations

**Summary pre-selection**: for gotcha nodes with `score ≥ 80`, the index entry's `summary` field may answer the query directly — no full node fetch needed.

**Zero cost**: pure HTTP GET against GitHub's CDN. No auth, no rate limits for reasonable usage, no infrastructure to maintain.

---

## Repository Structure

```
catalog.json                     ← root: all domains, hot nodes, total counts
domains/
└── {domain}/
    ├── index.json               ← node list with scores, summaries, types
    └── nodes/
        └── {slug}.md            ← individual knowledge node
tags/
└── {tag}.json                   ← inverted index: tag → list of node IDs + scores
signals/
└── scores.json                  ← precomputed scores (nightly refresh)
meta/
├── schema.json                  ← field definitions, edge types, score formula
└── templates/
    ├── gotcha.md
    ├── pattern.md
    ├── config.md
    ├── decision.md
    └── reference.md
submissions/
└── template.md                  ← contributor guide
.github/workflows/
├── validate-submission.yml      ← runs on PR: schema + conflict detection
├── rebuild-indexes.yml          ← runs on merge: atomic index rebuild
└── score-refresh.yml            ← nightly: recompute scores, mark stale nodes
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

## Scoring

Every node has a score from 0–100:

```
base: 50
+ vote component:    ( up / (up + down + 1) ) × 30      (max 30)
+ use component:     min( log10(uses+1) / log10(1000) × 20, 20 )  (max 20)
× staleness penalty: × 0.85 when TTL expires (fresh → false)
= 0 if superseded
```

High-score nodes appear in `hot_nodes` in catalog.json and are prioritized in retrieval.

---

## Contributing

**1. Pick a type.** Each node is exactly one of the 5 types. See [submissions/template.md](submissions/template.md) for type-specific templates and the full quality bar.

**2. Create the file:**
```
domains/{domain}/nodes/{slug}.md
```

**3. Open a PR** with title format:
```
[gotcha] ai-agents: Claude Code hooks hang without bypassPermissions
[pattern] auth: Supabase SSR session refresh with Next.js
[decision] database: Drizzle over Prisma for edge runtimes
```

**4. CI validates:**
- All required frontmatter fields present
- Type-specific fields valid (severity for gotcha, pattern_type for pattern, etc.)
- `id` field matches file path exactly
- `conflicts_with` edges declared on both sides
- On merge: domain indexes and catalog rebuilt automatically

**Quality bar in short:** specific (names exact methods/flags), reproducible (working example), scoped (states which versions), honest (admits when the alternative is fine), atomic (one failure mode or one decision per node).

---

## Node Format

Every node is a markdown file with YAML frontmatter. Example:

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
Hooks run without a TTY. Claude's permission system waits for stdin input that never arrives.

## Fix
```bash
nohup claude --agent kage-distiller --print "$TASK" \
  --permission-mode bypassPermissions \
  --no-session-persistence \
  >> distill.log 2>&1 &
```

## Never Do
```bash
# Missing --permission-mode → hangs forever
nohup claude --agent kage-distiller --print "$TASK" >> distill.log 2>&1 &
```

## Scope
All Claude Code versions ≥ 1.0. Affects Stop, PreToolUse, PostToolUse, and SessionStart hooks.
```

---

## Automation

Three GitHub Actions keep the graph healthy:

**`validate-submission`** — runs on every PR touching `domains/**/nodes/*.md`:
- Checks all required fields and type-specific fields
- Validates `id` matches file path
- Detects unidirectional `conflicts_with` edges (must be declared on both sides)

**`rebuild-indexes`** — runs on every merge to main:
- Rebuilds all `domains/*/index.json` files from node frontmatter
- Regenerates `catalog.json` with current counts and hot_nodes
- Rebuilds `tags/*.json` inverted indexes
- Commits as `[skip ci]`

**`score-refresh`** — runs nightly:
- Recomputes scores from current vote + use counts
- Marks nodes `fresh: false` when TTL expires, applies 0.85× staleness penalty
- Sets score to 0 for superseded nodes
- Writes `signals/scores.json`

---

## Edge Types

Nodes can declare relationships in their `related` array:

| Edge | Agent behavior |
|---|---|
| `requires` | Agent auto-fetches the linked node when fetching this one |
| `complements` | Cited in output only — not fetched |
| `alternative` | Cited with description — same problem, different approach |
| `supersedes` | This node is current; the linked node is outdated — agent redirects |
| `specializes` | Version-specific variant of a general node |
| `conflicts_with` | Cannot apply both — agent emits warning block. Must be bidirectional. |
