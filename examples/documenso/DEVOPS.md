---
devmd: devops
version: "1.0"
project: documenso
updated: 2026-05-13

build:
  system: turborepo
  bundler: vite
  container: "Docker multi-stage (4 stages)"
  node: "22-alpine"

ci:
  platform: github-actions
  workflows: 16
  e2e_runner: "ubuntu-latest-8-cores"

deploy:
  one_click: [railway, render, koyeb, elestio]
  custom: docker
---

# DEVOPS.md — Documenso

## Docker Multi-Stage Build

Four-stage build optimized for production image size and build caching.

### Stage 1: Base

```dockerfile
FROM node:22-alpine AS base
RUN apk add --no-cache libc6-compat
WORKDIR /app
```

Shared base image with system dependencies. Alpine for minimal size.

### Stage 2: Builder

```dockerfile
FROM base AS builder
COPY package.json package-lock.json turbo.json ./
COPY packages/*/package.json ./packages/
COPY apps/*/package.json ./apps/
RUN npm ci
COPY . .
RUN npx prisma generate
RUN npx turbo build --filter=@documenso/remix
```

Full dependency install + build. Turborepo caches intermediate results.

### Stage 3: Installer

```dockerfile
FROM base AS installer
WORKDIR /app
COPY --from=builder /app .
RUN npm ci --production
RUN npx prisma generate
```

Production-only dependencies. Strips devDependencies.

### Stage 4: Runner

```dockerfile
FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 documenso
COPY --from=installer --chown=documenso:nodejs /app/apps/remix/build ./build
COPY --from=installer --chown=documenso:nodejs /app/node_modules ./node_modules
COPY --from=installer --chown=documenso:nodejs /app/packages/prisma ./packages/prisma
USER documenso
EXPOSE 3000
CMD ["node", "build/server/index.js"]
```

Minimal runtime. Non-root user. Only build artifacts + production node_modules + Prisma client.

### Image Size

| Stage | Purpose | Approx. Size |
|---|---|---|
| Builder | Full source + devDeps | ~2 GB |
| Runner (final) | Runtime only | ~250 MB |

---

## Docker Compose Development Setup

```yaml
services:
  postgres:
    image: postgres:15
    ports:
      - "54320:5432"
    environment:
      POSTGRES_USER: documenso
      POSTGRES_PASSWORD: documenso
      POSTGRES_DB: documenso
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U documenso"]
      interval: 5s
      timeout: 5s
      retries: 5

  inbucket:
    image: inbucket/inbucket
    ports:
      - "9000:9000"    # Web UI (view caught emails)
      - "2500:2500"    # SMTP (application sends here)
      - "1100:1100"    # POP3

  minio:
    image: minio/minio
    ports:
      - "9001:9000"    # S3 API
      - "9002:9001"    # Console UI
    environment:
      MINIO_ROOT_USER: documenso
      MINIO_ROOT_PASSWORD: documenso
    command: server /data --console-address ":9001"
    volumes:
      - minio_data:/data

volumes:
  postgres_data:
  minio_data:
```

### Service Ports Reference

| Service | Port | URL | Purpose |
|---|---|---|---|
| PostgreSQL | 54320 | `postgresql://documenso:documenso@localhost:54320/documenso` | Application database |
| Inbucket Web | 9000 | `http://localhost:9000` | View caught test emails |
| Inbucket SMTP | 2500 | `smtp://localhost:2500` | Email sending target |
| MinIO S3 API | 9001 | `http://localhost:9001` | S3-compatible storage |
| MinIO Console | 9002 | `http://localhost:9002` | Storage admin UI |

### DevContainer

`.devcontainer/devcontainer.json` provides a ready-to-code environment:

- Node.js 22 base image
- PostgreSQL 15 service container
- Pre-installed extensions: Biome, Prisma
- Post-create command: `npm install && npx prisma migrate dev`

---

## GitHub Actions Workflows (16)

### CI / Quality

| Workflow | File | Trigger | Duration | Description |
|---|---|---|---|---|
| CI | `ci.yml` | PR + push to main | ~10 min | Lint + type-check + E2E tests (combined) |
| E2E | `e2e.yml` | PR | ~45 min | Full Playwright suite on 8-core runner |
| Lint | `lint.yml` | PR | ~2 min | Biome check across all packages |
| Type Check | `typecheck.yml` | PR | ~3 min | `tsc --noEmit` across all packages via Turborepo |
| CodeQL | `codeql.yml` | Schedule (weekly) + PR | ~15 min | GitHub security scanning (JS/TS) |
| Translations | `translations.yml` | PR | ~1 min | Verify i18n catalog completeness (11 languages) |

### Deployment

| Workflow | File | Trigger | Description |
|---|---|---|---|
| Deploy Preview | `deploy-preview.yml` | PR | Spin up preview environment per PR |
| Deploy Production | `deploy-production.yml` | Push to main | Build → push image → migrate DB → deploy → smoke test → notify Slack |
| Docker Build | `docker-build.yml` | Release tag | Build and push Docker images to container registry |
| Release | `release.yml` | Manual dispatch | Create GitHub release with auto-generated changelog |

### Automation

| Workflow | File | Trigger | Description |
|---|---|---|---|
| PR Auto-Label | `pr-auto-label.yml` | PR | Label PRs by changed file paths (e.g., `packages/prisma/**` → `prisma` label) |
| PR Title Check | `pr-title-check.yml` | PR | Verify PR title follows conventional commit format |
| Dependabot Auto | `dependabot-auto.yml` | Dependabot PR | Auto-merge minor/patch dependency updates after CI passes |
| Stale | `stale.yml` | Schedule (daily) | Mark issues/PRs stale after 30 days, close after 7 more |
| Lock Threads | `lock-threads.yml` | Schedule (weekly) | Lock resolved issue threads after 60 days |
| Contributor Welcome | `contributor-welcome.yml` | First PR by new contributor | Post welcome message with contribution guide links |

### E2E CI Configuration Detail

```yaml
jobs:
  e2e:
    runs-on: ubuntu-latest-8-cores
    timeout-minutes: 60
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: documenso_test
          POSTGRES_USER: documenso
          POSTGRES_PASSWORD: documenso
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'
      - run: npm ci
      - run: npx prisma migrate deploy
      - run: npx prisma db seed
      - run: npx playwright install --with-deps chromium
      - run: npx playwright test
        env:
          DATABASE_URL: postgresql://documenso:documenso@localhost:5432/documenso_test
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: packages/app-tests/playwright-report/
          retention-days: 7
```

---

## Turborepo Build Pipeline

### Task Graph

```
turbo.json tasks:
  build:
    dependsOn: [^build]         # Build dependencies first
    outputs: ["build/**"]
    
  typecheck:
    dependsOn: [^build]         # Need built packages for types
    
  lint:
    dependsOn: []               # Independent, runs in parallel
    
  dev:
    dependsOn: [^build]
    cache: false
    persistent: true
```

### Build Order

```
@documenso/prisma (generate)
        │
        ├── @documenso/lib
        │       │
        │       ├── @documenso/trpc
        │       ├── @documenso/email
        │       ├── @documenso/auth
        │       └── @documenso/signing
        │
        ├── @documenso/ui (independent, parallel)
        │
        └── @documenso/api
                │
                └── apps/remix (final)
```

### Caching

- **Local**: `.turbo/` directory. Hashes inputs (source files + env vars) → skips unchanged packages.
- **Remote** (optional): Vercel Remote Cache or custom server. Shares build cache across CI runs.

```bash
# Full build
npx turbo build                    # ~2 min cold, ~30s cached

# Selective build
npx turbo build --filter=@documenso/remix    # Only app + its deps

# Type-check all
npx turbo typecheck                # Parallel across all packages
```

---

## One-Click Deploy Targets

### Railway

[![Deploy on Railway](https://railway.app/button.svg)](https://railway.app/template/...)

- Pre-configured template with PostgreSQL + Redis add-ons
- Auto-detects Dockerfile
- Environment variables set via Railway dashboard
- Supports preview environments per PR

### Render

- `render.yaml` blueprint:
  - Web service: Docker build
  - PostgreSQL database
  - Redis (for BullMQ)
- Auto-deploy on push to main

### Koyeb

- Dockerfile-based deployment
- Built-in PostgreSQL via Neon integration
- Auto-scaling and health checks

### Elestio

- Fully managed Docker deployment
- Includes PostgreSQL, Redis, S3-compatible storage
- One-click setup with all environment variables pre-configured

---

## Deployment Checklist

### First Deploy

1. Provision PostgreSQL 15+ database
2. Set all required environment variables (see @CONFIG.md#core)
3. Choose and configure providers (see @CONFIG.md#provider-matrix)
4. Generate signing certificate (or configure HSM)
5. Run `npx prisma migrate deploy` against production database
6. Build and deploy Docker image
7. Verify health endpoint responds
8. Send test document to verify full flow

### Routine Deploy

1. CI passes (lint + typecheck + E2E)
2. Docker image built and pushed
3. Database migrations applied (`prisma migrate deploy`)
4. New container deployed (rolling update)
5. Smoke test: create document → send → sign → seal
6. Monitor error rates for 15 minutes
7. Rollback if error rate > 1% (see @OPERATIONS.md)

### Rollback Procedure

1. Revert to previous Docker image tag
2. If DB migration was applied and must be reverted:
   - `npx prisma migrate resolve --rolled-back <migration-name>`
   - Apply manual SQL to undo schema changes (migrations are forward-only by design)
3. Verify application health
4. Investigate root cause before re-deploying

## Cross-References

- Environment variables: @CONFIG.md
- Provider configuration: @INFRA.md#swappable-providers
- CI test details: @TESTING.md#ci-pipeline
- Production monitoring: @OPERATIONS.md
- Architecture (monorepo structure): @ARCHITECTURE.md
- Docker image security (non-root user): @SECURITY.md
