# Kage Graph — Contribution Guide

## Before You Submit

1. **Search first.** Run `/kage search <keyword>` or browse [catalog.json](../catalog.json). A duplicate PR will be closed.
2. **Pick one type.** Each node is exactly one of the 5 types below. Mixing types = rejected.
3. **One node per file.** File name = slug. Path = `domains/{domain}/nodes/{slug}.md`.
4. **No PII.** Strip all API keys, emails, user IDs, internal hostnames before submitting.

---

## Step 1 — Choose a Node Type

| Type | One-line definition | Signal words |
|---|---|---|
| `gotcha` | One failure mode. Symptom → Cause → Fix. Max 200 tokens. | "getting `[]`", "hangs", "error thrown", "silent", "crashes", "broken" |
| `pattern` | Multi-step implementation blueprint with working code. | "how to implement", "set up", "pattern for", "build" |
| `config` | Version-locked configuration block that must be exactly right. | "Dockerfile", "nginx", "tsconfig", ".env", "github-actions" |
| `decision` | Architectural trade-off frozen in time. X over Y and why. | "X vs Y", "should I use", "why not", "chose" |
| `reference` | Dense lookup table. API shapes, error codes, CLI flags. No prose. | "list of", "what are the events", "what fields", "error codes" |

---

## Step 2 — Copy the Right Template

### Template: `gotcha`

```markdown
---
id: "{domain}/{slug}"
type: "gotcha"
title: ""
domain: ""
tags: []
stack: ["lib@>=version"]
severity: "silent-failure"          # silent-failure | hard-error | data-corruption
score: 50
uses: 0
votes: {up: 0, down: 0}
added: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
fresh: true
ttl_days: 730
supersedes: null
superseded_by: null
related: []
---

# {Title}

## Symptom
What the developer observes. Be specific: "returns `[]`", "hangs on exit", "TypeError: X is not a function".

## Cause
One sentence. The root mechanism, not a description of the symptom.

## Fix
```bash
# Minimal working example
```

## Never Do
```bash
# What NOT to do, with a comment explaining why
```

## Scope
Which versions, environments, or conditions trigger this. One sentence.
```

---

### Template: `pattern`

```markdown
---
id: "{domain}/{slug}"
type: "pattern"
title: ""
domain: ""
tags: []
stack: ["lib@>=version"]
pattern_type: "integration"         # integration | auth-flow | data-pipeline | error-handling | caching | deployment
score: 50
uses: 0
votes: {up: 0, down: 0}
added: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
fresh: true
ttl_days: 365
supersedes: null
superseded_by: null
related: []
---

# {Title}

## When to Use
Specific conditions where this pattern applies. When NOT to use it.

## Required Setup
Dependencies, environment variables, or one-time configuration needed before the pattern works.

## Implementation
Working code. Real method names. No pseudocode.

```language
# Step 1: ...
# Step 2: ...
```

## Critical Gotchas
- What breaks if you skip step N
- Common misconfiguration

## Verification
How to confirm it's working. A test, a log line, a curl command.
```

---

### Template: `config`

```markdown
---
id: "{domain}/{slug}"
type: "config"
title: ""
domain: ""
tags: []
stack: ["tool@>=version"]
config_target: "Dockerfile"         # Dockerfile | nginx.conf | .env | tsconfig | vite.config | github-actions | package.json | next.config | other
score: 50
uses: 0
votes: {up: 0, down: 0}
added: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
fresh: true
ttl_days: 180
supersedes: null
superseded_by: null
related: []
---

# {Title}

## Requires
Minimum version, OS, or dependency assumptions.

## Config

```toml/yaml/json/dockerfile
# Full working configuration
# Every line that is non-obvious should have an inline comment
```

## What Breaks Without This
Specific error or behavior when this config is missing or wrong.
```

---

### Template: `decision`

```markdown
---
id: "{domain}/{slug}"
type: "decision"
title: ""
domain: ""
tags: []
stack: ["lib@>=version"]
decision_type: "library-choice"     # library-choice | architecture | hosting | data-model | api-style | infrastructure
decided_as_of: "YYYY-MM-DD"
score: 50
uses: 0
votes: {up: 0, down: 0}
added: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
fresh: true
ttl_days: 180
supersedes: null
superseded_by: null
related: []
---

# {Title}

## Decision
One sentence: "Use X over Y for Z."

## Why
The specific constraints that drove this choice. Not generic praise for X.

## When the Other Choice is Fine
The conditions under which Y is actually better. Be honest — this makes the node trustworthy.

## Migration Cost
If you switch away from X later: what's the blast radius? Data migration? API changes?

## Revise This Decision When
Concrete trigger conditions: "X raises prices above $Y/mo", "Next.js adds native support for Z", "team grows beyond N people".
```

---

### Template: `reference`

```markdown
---
id: "{domain}/{slug}"
type: "reference"
title: ""
domain: ""
tags: []
stack: ["service@>=version"]
reference_type: "event-catalog"     # event-catalog | error-codes | api-shape | cli-flags | env-vars | status-codes
score: 50
uses: 0
votes: {up: 0, down: 0}
added: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
fresh: true
ttl_days: 365
supersedes: null
superseded_by: null
related: []
---

# {Title}

| Key | Value | Notes |
|-----|-------|-------|
| ... | ...   | ...   |

## Commonly Missed
The entry or entries that developers get wrong most often. Call it out explicitly.
```

---

## Step 3 — Fill the `related` Array

```yaml
related:
  - id: "{domain}/{slug}"
    rel: "requires"       # requires | complements | alternative | supersedes | specializes | conflicts_with
```

| Edge | Use when |
|---|---|
| `requires` | Readers must understand the linked node before this one is actionable. Agent auto-fetches it. |
| `complements` | Linked node adds useful context but is not mandatory. Cited only. |
| `alternative` | Solves the same problem differently. Mutually exclusive. |
| `supersedes` | This node replaces an older node. Set `superseded_by` in the old node's frontmatter too. |
| `specializes` | This node is a version-specific or context-specific variant of a general node. |
| `conflicts_with` | Both nodes cannot be applied simultaneously. Must be declared on both sides or CI will reject. |

---

## Step 4 — File Placement

```
domains/{domain}/nodes/{slug}.md
```

- `domain`: one of `auth`, `database`, `deployment`, `frontend`, `testing`, `api-design`, `ai-agents`, `payments`, `storage`, `email`
- `slug`: kebab-case, lowercase, no special chars except hyphens. Max 60 chars.
- `id` field must equal `{domain}/{slug}` exactly — CI validates this.

---

## Step 5 — Open a PR

PR title format: `[{TYPE}] {domain}: {title}`

Examples:
- `[gotcha] ai-agents: Claude Code hooks hang without bypassPermissions`
- `[pattern] auth: Supabase SSR session refresh with Next.js App Router`
- `[decision] database: Drizzle over Prisma for edge runtimes`

The `validate-submission` workflow will check:
- All required frontmatter fields present
- Type-specific fields valid
- `id` matches file path
- `conflicts_with` edges declared on both sides
- Domain index updated (warning only — `rebuild-indexes` handles it on merge)

---

## Quality Bar

| Accepted | Rejected |
|---|---|
| Specific: names the exact method, flag, config key | Vague: "make sure you configure X properly" |
| Reproducible: includes a working example | Abstract: pattern without code |
| Scoped: states which versions it applies to | Blanket: "works with React" |
| Honest: admits when the alternative is fine | Opinionated without nuance |
| Atomic: one failure mode / one decision | Kitchen sink: multiple unrelated things |

Nodes that are too large will be asked to be split. Nodes that are too vague will be closed.
