---
id: "deployment/docker-env-not-in-image"
type: "gotcha"
title: "Secrets baked into Docker image layers are recoverable even after removal"
domain: "deployment"
tags: ["docker", "secrets", "environment", "build", "security", "layers"]
stack: ["docker@>=20"]
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
related: []
summary: "Secrets written in a Dockerfile layer survive even if a later RUN removes them — every layer is recoverable via docker history or image export. Use BuildKit --secret mounts, runtime ENV vars, or multi-stage builds where the build stage never ships."
---

# Secrets baked into Docker image layers are recoverable even after removal

## Symptom
CI builds pass secrets via `ARG`/`ENV` or COPY of `.env` file. A later `RUN rm .env` appears to clean it up. But the secret is still present in intermediate layers and recoverable via `docker history`, `docker save | tar`, or image analysis tools.

## Cause
Each Docker instruction creates an immutable layer. A secret written in layer N is part of that layer's content even if layer N+1 deletes it. Anyone with image access (Docker Hub, ECR, registry) can extract all layers and read the secret.

## Fix

**Option A — Build-time secrets via `--secret` mount (Docker BuildKit):**
```dockerfile
# syntax=docker/dockerfile:1
FROM node:20-alpine AS builder

# Secret is mounted only for this RUN — never written to a layer
RUN --mount=type=secret,id=npmrc,dst=/root/.npmrc \
    npm ci

FROM node:20-alpine
COPY --from=builder /app/node_modules ./node_modules
```

```bash
# Build command — secret comes from file, never in Dockerfile
DOCKER_BUILDKIT=1 docker build \
  --secret id=npmrc,src=$HOME/.npmrc \
  -t myapp .
```

**Option B — Runtime secrets via environment variables (no baking):**
```dockerfile
# Dockerfile has no secrets — only declares the variable
ENV DATABASE_URL=""

# Set at runtime via docker run, Kubernetes secret, or ECS task definition
```
```bash
docker run -e DATABASE_URL="$DATABASE_URL" myapp
```

**Option C — Multi-stage build (credential lives only in build stage):**
```dockerfile
FROM node:20-alpine AS fetcher
ARG NPM_TOKEN            # only exists in this stage
RUN echo "//registry.npmjs.org/:_authToken=$NPM_TOKEN" > .npmrc && \
    npm ci && \
    rm .npmrc            # safe — stage is not shipped

FROM node:20-alpine AS final
COPY --from=fetcher /app/node_modules ./node_modules
# NPM_TOKEN never present in final image
```

## Never Do
```dockerfile
# Secret is in layer history forever
ARG DATABASE_URL
ENV DATABASE_URL=$DATABASE_URL

# Removing in a later layer doesn't help
COPY .env .
RUN npm run build
RUN rm .env   # too late — .env is already in a layer
```

## Scope
Affects any Docker image built with `docker build` (without BuildKit secrets). Final images pushed to any registry — public or private — expose all layer content to anyone with pull access. Use `docker history --no-trunc <image>` to confirm a secret was baked in.
