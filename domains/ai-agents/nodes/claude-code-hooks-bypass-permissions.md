---
id: "ai-agents/claude-code-hooks-bypass-permissions"
title: "Claude Code hooks must use --permission-mode bypassPermissions for sub-agents"
domain: "ai-agents"
tags: ["claude-code", "hooks", "agents", "permissions", "stop-hook", "background"]
stack: ["claude-code@1.x+"]
score: 90
uses: 1
votes: { up: 1, down: 0 }
added: "2026-04-12"
updated: "2026-04-12"
fresh: true
ttl_days: 730
supersedes: null
superseded_by: null
related: []
---

# Claude Code hooks must use --permission-mode bypassPermissions for sub-agents

## Problem

When a Claude Code sub-agent is invoked from a hook (Stop, PreToolUse, PostToolUse, etc.), the hook runs in a non-interactive background context with no TTY. If the sub-agent attempts to prompt for user permission, the process hangs indefinitely — no one is there to respond.

## Solution

Always pass `--permission-mode bypassPermissions` when launching a sub-agent from any hook context:

```bash
nohup claude \
  --agent kage-distiller \
  --print "$TASK" \
  --permission-mode bypassPermissions \
  --no-session-persistence \
  >> distill.log 2>&1 &
```

## Applies To

- Stop hooks
- PreToolUse / PostToolUse hooks
- SessionStart hooks
- Any `nohup ... &` or background subprocess that calls `claude --agent`

## Why

Hooks execute as shell subprocesses launched by Claude Code. They have no associated terminal session. Claude's permission prompts write to stdout and wait for stdin — which is never provided in this context, causing the process to block forever and eventually time out, silently failing the hook.

## Gotcha

`--no-session-persistence` should also be passed for background distiller invocations to prevent the distiller session from being saved to `~/.claude/projects/`, which would trigger another Stop hook cycle.
