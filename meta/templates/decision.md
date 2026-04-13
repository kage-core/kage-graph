---
id: "{domain}/{slug}"
type: "decision"
title: "X over Y for Z"
domain: ""
tags: []
stack: []
decision_type: "library-choice"
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

# {X over Y for Z}

## Decision
One sentence: use X when [condition]. Use Y when [other condition].

## Why
Technical reason X beats Y in this context. Be specific — benchmark numbers, API limitations, binary constraints.

## When Y is Fine
Conditions under which the alternative is actually the right call.

## Migration Cost
If switching from Y to X mid-project: what it takes (hours, what changes).

## Revise This Decision When
Specific future event that would make this decision wrong (e.g., library releases stable edge support, version X drops).
