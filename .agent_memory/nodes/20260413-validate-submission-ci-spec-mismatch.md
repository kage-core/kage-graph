---
title: "Fix: validate-submission.yml CI required non-existent fields and invalid severity values"
category: debugging
tags: ["ci", "validation", "spec-mismatch", "regression"]
paths: ".github/workflows"
date: "2026-04-13"
source: "kage-distiller"
session: "unknown"
---

# Fix: validate-submission.yml CI required non-existent fields and invalid severity values

## Problem

The `validate-submission.yml` GitHub Actions workflow had two critical validation bugs that would auto-reject any PR submission:

1. **Non-existent required fields:** The workflow required frontmatter fields `pattern_type`, `config_target`, `decision_type`, and `decided_as_of` that were never defined in the node spec or documented in README/template. Every existing pattern/config/decision node in the repo lacked these fields.

2. **Invalid severity enum:** The workflow validated for severity values `silent-failure` and `data-corruption`, which don't match what nodes actually use (`hard-error`, `warning`, `info`).

## Root Cause

CI validation rules were written without deriving them from the actual node specification or checking against existing nodes in the repository. The fields and values were invented rather than reverse-engineered from the spec.

## Solution

- Removed validation for `pattern_type`, `config_target`, `decision_type`, and `decided_as_of` (not part of spec)
- Corrected severity enum to match actual node values: `hard-error`, `warning`, `info`
- File: `.github/workflows/validate-submission.yml`

## Lesson

When writing CI validation for a schema:
1. **Derive rules from the actual spec** — read the node schema definition or template
2. **Validate against existing nodes** — check that real nodes in the repo satisfy your rules
3. **Don't invent requirements** — if no existing node uses a field, it's not required

This prevents creating validation logic that rejects everything, including the entire existing repository.
