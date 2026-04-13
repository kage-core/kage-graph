---
id: "ai-agents/claude-code-no-session-persistence"
type: "gotcha"
title: "Claude Code background agents must pass --no-session-persistence to avoid recursive Stop hooks"
domain: "ai-agents"
tags: ["claude-code", "hooks", "stop-hook", "session", "background", "infinite-loop"]
stack: ["claude-code@>=1.0"]
severity: "hard-error"
score: 50
uses: 0
votes: {up: 0, down: 0}
added: "2026-04-13"
updated: "2026-04-13"
fresh: true
ttl_days: 730
supersedes: null
superseded_by: null
related:
  - id: "ai-agents/claude-code-hooks-bypass-permissions"
    rel: "requires"
---

# Claude Code background agents must pass --no-session-persistence to avoid recursive Stop hooks

## Symptom
Stop hook fires, launches a background `claude --agent` process, which completes — then the Stop hook fires again. This repeats indefinitely, spawning new distiller processes on every cycle. Log file grows unbounded.

## Cause
When a `claude` subprocess completes, Claude Code saves its session to disk by default. That saved session triggers another Stop hook event. Each Stop hook spawns another agent, which saves another session, which fires another Stop hook.

## Fix
Always pass `--no-session-persistence` when invoking any sub-agent from a hook:

```bash
nohup claude \
  --agent kage-distiller \
  --print "$TASK" \
  --permission-mode bypassPermissions \
  --no-session-persistence \
  >> ~/.claude/kage/distill.log 2>&1 &
```

## Never Do
```bash
# Missing --no-session-persistence → session saved → Stop hook fires again → infinite loop
nohup claude \
  --agent kage-distiller \
  --print "$TASK" \
  --permission-mode bypassPermissions \
  >> distill.log 2>&1 &
```

## Scope
Affects any `claude` subprocess launched from a Stop hook. The loop compounds quickly — each cycle adds one more background process. Applies to all Claude Code versions ≥ 1.0.
