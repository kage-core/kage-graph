---
id: "ai-agents/claude-code-hooks-bypass-permissions"
type: "gotcha"
title: "Claude Code hooks must use --permission-mode bypassPermissions for sub-agents"
domain: "ai-agents"
tags: ["claude-code", "hooks", "agents", "permissions", "stop-hook", "background"]
stack: ["claude-code@>=1.0"]
severity: "hard-error"
score: 67
uses: 1
votes: {up: 1, down: 0}
added: "2026-04-12"
updated: "2026-04-12"
fresh: true
ttl_days: 730
supersedes: null
superseded_by: null
related:
  - id: "ai-agents/claude-code-no-session-persistence"
    rel: "requires"
summary: "Hooks run without a TTY — any permission prompt hangs indefinitely. Always pass --permission-mode bypassPermissions when invoking sub-agents from Stop/PreToolUse/PostToolUse hooks. Also pass --no-session-persistence to prevent recursive Stop hook firing."
---

# Claude Code hooks must use --permission-mode bypassPermissions for sub-agents

## Symptom
Hook fires, `claude --agent` process launches, then hangs indefinitely. The Stop hook times out (default 15s), Claude Code session appears to freeze on exit. Background distiller log shows no output or is empty.

## Cause
Hooks execute as shell subprocesses with no associated TTY. Claude's permission system writes prompts to stdout and waits for stdin — which is never provided in this context. The process blocks forever waiting for a response that will never come.

## Fix
Always pass `--permission-mode bypassPermissions` when invoking any sub-agent from a hook:

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
# Missing --permission-mode → hangs on first file write or tool use
nohup claude --agent kage-distiller --print "$TASK" >> distill.log 2>&1 &

# Missing --no-session-persistence → distiller session saved to disk,
# triggers another Stop hook, causes infinite loop
claude --agent kage-distiller --print "$TASK" --permission-mode bypassPermissions
```

## Scope
Affects: Stop hooks, PreToolUse hooks, PostToolUse hooks, SessionStart hooks, any `nohup ... &` subprocess calling `claude --agent`. Applies to all Claude Code versions that have the permission system (all >=1.0).
