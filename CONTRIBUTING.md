# Contributing to the Kage Knowledge Graph

## Who Can Submit

Anyone. Fork the repo, add a node file, open a PR. No account required beyond a GitHub login.

The easiest path is `/kage submit <node-file>` from inside Claude Code — it prepares the node, validates it, and opens the PR automatically if you have a fork.

---

## Who Reviews

**Maintainers** (kage-core team) review all PRs initially. As the community grows, frequent contributors with merged nodes are added as domain reviewers.

**Domain reviewers** are contributors with demonstrated expertise in a specific domain. They can approve PRs in their domain without a maintainer sign-off.

Every PR needs **1 approval + CI green** to merge. No exceptions.

---

## What Gets Reviewed

Reviewers check three things:

**1. Is it true?**
- Did you hit this problem yourself?
- Does the fix actually work on the stated stack versions?
- Is the scope accurate — does it affect all versions listed, or just some?

**2. Is it atomic?**
- One failure mode, one decision, one config block. Not a tutorial.
- If it covers more than one thing, ask to split it.

**3. Is it specific enough to be useful?**
- Names exact method names, flags, config keys, error messages
- Includes a working example (for gotcha/pattern/config)
- States which versions it applies to

If all three pass, the PR merges. Style opinions are not grounds for rejection.

---

## What Gets Rejected

- Vague advice ("make sure you handle errors properly")
- Duplicates of an existing node — check `/kage search` or browse the domain index first
- Missing `stack` field — nodes without version scope go stale and mislead
- Untested fixes — if you haven't run this yourself, it doesn't belong here
- PII, credentials, internal hostnames

---

## Updating an Existing Node

Edit the existing file directly — same path, same PR flow. Bump the `updated` field and reset `fresh: true`. Do not create a new file for a content update.

Use `supersedes`/`superseded_by` only when the underlying approach changes fundamentally, not for minor corrections.

---

## Becoming a Domain Reviewer

Open an issue titled "Reviewer request: {domain}" linking to your merged contributions in that domain. The kage-core team will add you.

There's no minimum contribution count — one high-quality merged node in a domain is enough to ask.
