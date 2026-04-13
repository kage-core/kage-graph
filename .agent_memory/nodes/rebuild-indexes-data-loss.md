---
title: "Index regeneration silently drops hand-authored fields"
category: architecture
tags: ["workflow", "data-loss", "index-generation"]
paths: ".github/workflows"
date: "2026-04-13"
source: "kage-distiller"
session: "unknown"
---

# Index regeneration silently drops hand-authored fields

## Problem

The `rebuild-indexes.yml` workflow regenerates all `domain/index.json` files from scratch on every push. Any field present in a hand-authored index.json but not explicitly extracted by the workflow script gets silently dropped — including `stack`, `curated summary`, and other metadata.

This creates a hidden data-loss trap: edits to `index.json` are overwritten, giving false impression that the index is editable when it's actually derived.

## Solution

Encode any curated data you want preserved back into the **source `.md` node files**, not the generated `index.json`:

1. **Add `summary` frontmatter field** to node `.md` files — the workflow checks for this field and uses it if present, otherwise falls back to extraction from content.

2. **Add `stack` to the workflow's node output dict** in the Python extraction script so stack data flows through to the generated index.

## General Rule

**Auto-generated files always win over hand edits.** Treat generated files as read-only. Any metadata that must survive regeneration must be:
- Stored in the source `.md` frontmatter
- Explicitly extracted by the workflow script into the output dict

This prevents silent data loss and makes the source of truth explicit.

## Related Files

- `.github/workflows/rebuild-indexes.yml` — workflow that regenerates indexes
- Source `.md` node files in each domain directory
- Python extraction script within the workflow
