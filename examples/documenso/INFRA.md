---
devmd: infra
version: "1.0"
project: documenso
updated: 2026-05-13

runtime: "Node.js 22"
container: "Docker multi-stage (Alpine)"
orchestration: "docker-compose (dev), varies (production)"

services:
  primary:
    - name: web
      image: "documenso/documenso"
      port: 3000
      description: "Main application — React Router v7 + Hono"

  dependencies:
    - name: postgresql
      version: "15"
      port_dev: 54320
      port_prod: 5432
      description: "Primary database"

    - name: inbucket
      port: 9000
      description: "Email testing server (dev only)"

    - name: minio
      ports: [9001, 9002]
      description: "S3-compatible object storage (dev only)"

swappable_providers:
  storage:
    - name: database
      description: "Store PDFs as base64 in DocumentData table"
      use_case: "Simple self-hosting, small deployments"
      env: "NEXT_PRIVATE_UPLOAD_TRANSPORT=database"
    - name: s3
      description: "Store PDFs in S3-compatible storage (AWS S3, MinIO, R2)"
      use_case: "Production, large deployments"
      env: "NEXT_PRIVATE_UPLOAD_TRANSPORT=s3"

  signing:
    - name: local
      description: "PKCS#12 (P12) certificate on filesystem"
      use_case: "Self-hosting, development"
      env: "NEXT_PRIVATE_SIGNING_TRANSPORT=local"
    - name: gcloud-hsm
      description: "Google Cloud Hardware Security Module"
      use_case: "Production, compliance-critical"
      env: "NEXT_PRIVATE_SIGNING_TRANSPORT=gcloud-hsm"

  email:
    - name: smtp-auth
      description: "SMTP with username/password authentication"
      env: "NEXT_PRIVATE_SMTP_TRANSPORT=smtp-auth"
    - name: smtp-api
      description: "SMTP with API key authentication"
      env: "NEXT_PRIVATE_SMTP_TRANSPORT=smtp-api"
    - name: resend
      description: "Resend email API"
      env: "NEXT_PRIVATE_SMTP_TRANSPORT=resend"
    - name: mailchannels
      description: "MailChannels email API"
      env: "NEXT_PRIVATE_SMTP_TRANSPORT=mailchannels"

  jobs:
    - name: local
      description: "Database-backed polling queue"
      use_case: "Simple self-hosting"
      env: "NEXT_PRIVATE_JOBS_TRANSPORT=local"
    - name: bullmq
      description: "Redis-backed BullMQ queue"
      use_case: "High throughput, reliability"
      env: "NEXT_PRIVATE_JOBS_TRANSPORT=bullmq"
      requires: redis
    - name: inngest
      description: "Inngest event-driven functions"
      use_case: "Serverless, managed infrastructure"
      env: "NEXT_PRIVATE_JOBS_TRANSPORT=inngest"

one_click_deploy:
  - platform: Railway
    url: "https://railway.app/template/documenso"
  - platform: Render
    url: "https://render.com/deploy?repo=documenso/documenso"
  - platform: Koyeb
    url: "https://www.koyeb.com/deploy/documenso"
  - platform: Elestio
    url: "https://elest.io/open-source/documenso"

env_var_count: "150+"
---

# INFRA.md — Documenso

## Docker Setup

### Production Image

Multi-stage Docker build based on Node.js 22 Alpine:

```dockerfile
# Stage 1: Install dependencies
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
COPY packages/*/package.json ./packages/
COPY apps/*/package.json ./apps/
RUN npm ci --production=false

# Stage 2: Build
FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npx prisma generate
RUN npx turbo build --filter=@documenso/remix

# Stage 3: Production runtime
FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/apps/remix/build ./build
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/packages/prisma ./packages/prisma
EXPOSE 3000
CMD ["node", "build/server/index.js"]
```

### Development Docker Compose

```yaml
# docker-compose.yml
version: '3.8'
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

  inbucket:
    image: inbucket/inbucket
    ports:
      - "9000:9000"    # Web UI
      - "2500:2500"    # SMTP
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

### DevContainer

`.devcontainer/devcontainer.json` provides a ready-to-code environment:

- Node.js 22
- PostgreSQL 15 service
- Pre-installed: Biome extension, Prisma extension
- Auto-runs `npm install` and `prisma migrate dev` on creation

## Swappable Providers

The provider abstraction is a core architectural pattern. Each provider implements the same interface, swapped via environment variable.

### Storage Provider Interface

```typescript
interface StorageProvider {
  upload(file: Buffer, key: string): Promise<string>;
  download(key: string): Promise<Buffer>;
  delete(key: string): Promise<void>;
  getUrl(key: string): Promise<string>;
}
```

| Provider | Env Value | Best For |
|---|---|---|
| Database | `database` | Simple self-hosting (<100 docs) |
| S3 | `s3` | Production (any S3-compatible: AWS, MinIO, R2, DigitalOcean Spaces) |

### Signing Provider Interface

```typescript
interface SigningProvider {
  sign(pdf: Buffer, certificate: CertificateInfo): Promise<Buffer>;
  getCertificateInfo(): Promise<CertificateInfo>;
}
```

| Provider | Env Value | Best For |
|---|---|---|
| Local P12 | `local` | Self-hosting, development |
| Google HSM | `gcloud-hsm` | Compliance, production |

### Email Provider Interface

```typescript
interface EmailProvider {
  send(options: EmailOptions): Promise<void>;
}
```

| Provider | Env Value | Configuration |
|---|---|---|
| SMTP Auth | `smtp-auth` | Host, port, username, password |
| SMTP API | `smtp-api` | Host, port, API key header |
| Resend | `resend` | API key |
| MailChannels | `mailchannels` | API key |

### Job Provider Interface

```typescript
interface JobProvider {
  enqueue(job: JobDefinition): Promise<void>;
  process(handler: JobHandler): void;
  schedule(cron: string, job: JobDefinition): void;
}
```

| Provider | Env Value | Trade-offs |
|---|---|---|
| Local | `local` | No dependencies, polling-based, single instance only |
| BullMQ | `bullmq` | Redis dependency, reliable, multi-instance, retry logic |
| Inngest | `inngest` | External dependency, serverless-friendly, event-driven |

See @RUNTIME.md for job execution details.

## CI/CD

### GitHub Actions (16 workflows)

See @TESTING.md#github-actions-workflows-16 for the full list.

Key deployment workflows:

```yaml
# deploy-production.yml
on:
  push:
    branches: [main]
jobs:
  deploy:
    steps:
      - Build Docker image
      - Push to container registry
      - Run database migrations
      - Deploy to production cluster
      - Run smoke tests
      - Notify Slack on success/failure
```

### Deployment Strategy

- **Staging**: Auto-deploy on PR merge to `main`
- **Production**: Auto-deploy after staging smoke tests pass
- **Rollback**: Previous Docker image tag, `prisma migrate resolve` for DB

## Environment Variables

150+ environment variables organized by category:

### Core

```bash
NEXT_PUBLIC_WEBAPP_URL=https://app.documenso.com
NEXT_PRIVATE_DATABASE_URL=postgresql://user:pass@host:5432/documenso
NEXT_PRIVATE_DIRECT_DATABASE_URL=postgresql://user:pass@host:5432/documenso
NEXTAUTH_SECRET=<random-32-bytes>
```

### Provider Selection

```bash
NEXT_PRIVATE_UPLOAD_TRANSPORT=s3|database
NEXT_PRIVATE_SIGNING_TRANSPORT=local|gcloud-hsm
NEXT_PRIVATE_SMTP_TRANSPORT=smtp-auth|smtp-api|resend|mailchannels
NEXT_PRIVATE_JOBS_TRANSPORT=local|bullmq|inngest
```

### S3 Storage

```bash
NEXT_PRIVATE_UPLOAD_BUCKET=documenso-documents
NEXT_PRIVATE_UPLOAD_REGION=us-east-1
NEXT_PRIVATE_UPLOAD_ENDPOINT=https://s3.amazonaws.com
NEXT_PRIVATE_UPLOAD_ACCESS_KEY_ID=
NEXT_PRIVATE_UPLOAD_SECRET_ACCESS_KEY=
```

### Signing

```bash
# Local
NEXT_PRIVATE_SIGNING_LOCAL_FILE_PATH=/certs/signing.p12
NEXT_PRIVATE_SIGNING_LOCAL_FILE_CONTENTS=<base64-encoded-p12>
NEXT_PRIVATE_SIGNING_PASSPHRASE=

# Google HSM
NEXT_PRIVATE_SIGNING_GCLOUD_HSM_KEY_PATH=
NEXT_PRIVATE_SIGNING_GCLOUD_APPLICATION_CREDENTIALS_CONTENTS=
```

### Email

```bash
NEXT_PRIVATE_SMTP_HOST=smtp.example.com
NEXT_PRIVATE_SMTP_PORT=587
NEXT_PRIVATE_SMTP_USERNAME=
NEXT_PRIVATE_SMTP_PASSWORD=
NEXT_PRIVATE_SMTP_FROM_ADDRESS=noreply@documenso.com
NEXT_PRIVATE_SMTP_FROM_NAME=Documenso
```

### Jobs (BullMQ)

```bash
NEXT_PRIVATE_REDIS_URL=redis://localhost:6379
```

### Auth

```bash
NEXT_PRIVATE_GOOGLE_CLIENT_ID=
NEXT_PRIVATE_GOOGLE_CLIENT_SECRET=
NEXT_PRIVATE_OIDC_WELL_KNOWN=
NEXT_PRIVATE_OIDC_CLIENT_ID=
NEXT_PRIVATE_OIDC_CLIENT_SECRET=
```

### Observability

```bash
NEXT_PUBLIC_POSTHOG_KEY=
NEXT_PRIVATE_DATADOG_API_KEY=
```

### Billing

```bash
NEXT_PRIVATE_STRIPE_API_KEY=
NEXT_PRIVATE_STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_COMMUNITY_PLAN_MONTHLY_PRICE_ID=
```

### Captcha

```bash
NEXT_PUBLIC_TURNSTILE_SITE_KEY=
NEXT_PRIVATE_TURNSTILE_SECRET_KEY=
```

## Cross-References

- Background job runtime: @RUNTIME.md
- Docker build context: @ARCHITECTURE.md
- CI pipeline details: @TESTING.md#ci-pipeline
- Security of infrastructure: @SECURITY.md
- Dev service ports: @CLAUDE.md#dev-services
