---
id: "tooling/shell-env-var-inline-script"
type: "gotcha"
title: "Shell variables with special characters break inline scripts — use env var pattern"
domain: "tooling"
tags: ["shell", "bash", "python", "string-interpolation", "quoting", "inline-script"]
stack: ["bash", "zsh", "python3"]
severity: "hard-error"
score: 50
uses: 0
votes: {up: 0, down: 0}
added: "2026-04-18"
updated: "2026-04-18"
fresh: true
ttl_days: 730
supersedes: null
superseded_by: null
related: []
---

# Shell variables with special characters break inline scripts — use env var pattern

## Symptom

Inline script fails with a syntax error when a shell variable contains an apostrophe, quote, backslash, or Unicode.

```bash
# Fails if $MSG contains "don't" or any apostrophe
python3 -c "import json; print(json.dumps({'key': '$MSG'}))"
# SyntaxError: unterminated string literal
```

## Cause

The apostrophe in `$MSG` terminates the Python string literal before the script closes. The shell expands the variable before Python sees it, so quoting rules of both languages collide.

## Fix

Pass the variable through an environment variable — keeps it entirely outside the inline script string:

```bash
MY_VAR="$MSG" python3 -c "import json,os; print(json.dumps({'key': os.environ['MY_VAR']}))"
```

Works for any special character: apostrophes, double quotes, backslashes, newlines, Unicode.

## Never Do

```bash
# Never interpolate shell vars directly into inline script strings
node -e "console.log('$VAR')"
ruby -e "puts '$VAR'"
python3 -c "print('$VAR')"
```

## Scope

Applies to bash/zsh inline scripts calling any interpreter (Python, Node, Ruby, Perl). Use environment variables as the default pattern whenever the variable could contain user-supplied or freeform text.
