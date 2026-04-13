---
title: "Bash JSON generation via string interpolation is unsafe—use Python or jq"
category: debugging
tags: ["json", "bash", "workflows", "rebuild-indexes"]
paths: ".github/workflows"
date: "2026-04-13"
source: "kage-distiller"
session: "unknown"
---

# Bash JSON generation via string interpolation is unsafe—use Python or jq

## Problem

The `rebuild-indexes.yml` workflow step that built JSON via bash string concatenation produced a `JSONDecodeError`. Root cause: bash interpolation does not safely handle special characters found in real content.

### Specific failures:

- **Backticks (`)** in content are interpreted as subshell commands
- **Em-dashes, curly quotes, and Unicode** corrupt the output structure
- **Long strings** cause silent truncation
- **Unescaped quotes and newlines** break JSON syntax

Example: `json_str="{ \"value\": \"$content\" }"` fails when `$content` contains `{key: value}` or backticks.

## Solution

Rewrite any workflow step that builds JSON using:

1. **Python** with `json.dump()` — preferred, most reliable
2. **jq** — acceptable for simple transformations

Both handle escaping, encoding, and special characters correctly.

## Implementation in rebuild-indexes.yml

Replace the bash string concatenation step entirely with a Python script or separate Python step using `json.dump()` to write to stdout or a file.

## Rule

**Any workflow step that builds JSON must use Python's `json.dump()` or `jq`, never bash string interpolation.**

Check all `.yml` files for patterns like:
```bash
json_str="{\"key\": \"$var\"}"
```

Replace with Python or jq.
