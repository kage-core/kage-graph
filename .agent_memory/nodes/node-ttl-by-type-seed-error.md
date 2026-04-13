---
title: "kage-graph node TTL values by type—corrected seed values"
category: config
tags: ["node-spec", "ttl", "seeding", "data-quality"]
paths: "kage-graph"
date: "2026-04-13"
source: "kage-distiller"
session: "unknown"
---

# kage-graph node TTL values by type—corrected seed values

## The Problem

The initial 5 gotcha seed nodes were created with `ttl_days: 365` (1 year), but the node spec defines gotcha type with `ttl_days: 730` (2 years). Similarly, 1 config seed node had `ttl_days: 365` when config type should be `180` days.

Reason: gotchas change slowly (2-year expiry makes sense for environmental gotchas); config APIs and patterns iterate faster (180–365 days).

## Correct TTL Mapping by Node Type

When seeding or creating new nodes, always use:

| Node Type | TTL (days) | Rationale |
|-----------|-----------|-----------|
| `gotcha` | 730 | Environmental gotchas are slow-moving; 2-year window |
| `config` | 180 | Config APIs change faster |
| `decision` | 180 | Architecture decisions may need refresh as deps evolve |
| `pattern` | 365 | Patterns are relatively stable |
| `reference` | 365 | Reference docs stable within a year |

## Action

Audit all existing seed nodes and correct `ttl_days` to match the type. Update the seeding code to always assign TTL based on the node type, not default to 365 for all types.
