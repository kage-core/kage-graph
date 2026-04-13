---
title: "Default branch is master, not main — broke workflows and CDN URLs"
category: repo_context
tags: ["github", "workflows", "configuration", "cdn"]
paths: ".github/workflows"
date: "2026-04-13"
source: "kage-distiller"
session: "unknown"
---

# Default branch is master, not main — broke workflows and CDN URLs

## Problem

The kage-graph repository uses `master` as its default branch, not `main`. This caused two critical issues:

1. **GitHub Actions workflow never fired**: The `rebuild-indexes.yml` workflow had `branches: [main]` trigger, so it never executed when code was pushed to `master`.

2. **CDN URL 404s**: All CDN URLs in `kage-graph.md` and `session-start.sh` referenced `/main/` paths, causing 404 errors on every agent fetch.

## Root Cause

Assumption that the repository follows the GitHub default naming convention (`main`). This is not always true — some repos still use `master` as their default branch.

## Solution

1. In `.github/workflows/rebuild-indexes.yml`: Change `branches: [main]` to `branches: [master]`
2. In `kage-graph.md`: Update all CDN URLs from `/main/` to `/master/`
3. In `session-start.sh`: Update all CDN URLs from `/main/` to `/master/`

## Lesson

Always verify the actual default branch name before:
- Configuring GitHub Actions triggers
- Building CDN URLs
- Writing documentation that references branch paths

Use `git branch -a` or check GitHub repo settings to confirm.
