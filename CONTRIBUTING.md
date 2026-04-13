# Contributing to the Kage Knowledge Graph

## How It Works

```
/kage submit <node-file>
        │
        ▼
PR opened on kage-core/kage-graph
        │
        └── validate-submission (CI)
            ├── schema: required fields, type-specific fields, id matches path
            ├── slug collision: new slug doesn't already exist on main
            └── conflict edges: conflicts_with declared on both sides
                    │
                    CI passes
                    │
                    ▼
            maintainer reviews content → approves → merge
                    │
                    ▼
            rebuild-indexes runs → live on CDN in minutes
```

---

## Submitting

From Claude Code:
```
/kage submit .agent_memory/nodes/my-node.md
```

This validates the node locally, adds the required global fields (`type`, `id`, `score`, `ttl_days`), and opens a PR. If you don't have a fork, it saves the prepared node locally and gives you the manual steps.

Manually: fork the repo, create `domains/{domain}/nodes/{slug}.md`, open a PR with title `[{type}] {domain}: {title}`.

---

## What the Review Bot Checks

The Claude-powered review bot approves automatically if all of these are true:

| Check | What it means |
|---|---|
| **Specific** | Names exact method names, CLI flags, config keys, or error messages — not vague advice |
| **Atomic** | Covers exactly one failure mode, one decision, or one config block |
| **Scoped** | `stack` field states which versions this applies to |
| **Example** | For gotcha/pattern/config: includes a working code example |
| **Honest** | For decision: acknowledges when the alternative is fine |
| **No PII** | No API keys, passwords, emails, internal hostnames |

If any check fails, the bot leaves specific feedback on the PR. Fix the issues, push a new commit — the bot re-reviews automatically.

---

## Updating an Existing Node

Edit the existing file directly. Bump `updated`, reset `fresh: true`. Same PR flow, same bot review. Do not create a new file for a content update — the slug collision check will block it.

## Superseding a Node

When the underlying approach changes fundamentally, create a new node and link both sides:

```yaml
# new node frontmatter
supersedes: "domain/old-slug"

# old node frontmatter (edit in the same PR)
superseded_by: "domain/new-slug"
```

The agent silently redirects from old to new at retrieval time. The old node's score drops to 0 on the next nightly refresh.

---

## What Gets Rejected

- Vague advice without specific method/flag/config names
- No working example for gotcha, pattern, or config types
- Missing or vague `stack` field (e.g. `stack: ["react"]` instead of `stack: ["react@>=18"]`)
- Node covers multiple unrelated things — split it
- Duplicate of an existing node — check `/kage search` first
- PII, credentials, internal hostnames
