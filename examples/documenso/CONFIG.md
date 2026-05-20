---
devmd: config
version: "1.0"
project: documenso
updated: 2026-05-13

total_env_vars: "150+"
validation: "Zod schemas at startup"
prefix_convention:
  public: "NEXT_PUBLIC_"
  private: "NEXT_PRIVATE_"
access_pattern: "packages/lib/constants/ — never process.env directly"

providers:
  storage: [database, s3]
  signing: [local, gcloud-hsm]
  email: [smtp-auth, smtp-api, resend, mailchannels]
  jobs: [local, bullmq, inngest]
---

# CONFIG.md — Documenso

## Environment Variable Reference

All environment variables are validated via Zod at application startup. Invalid or missing required vars cause immediate failure with descriptive error messages.

**Access pattern**: Always import from `packages/lib/constants/` — never read `process.env` directly in application code. See @CLAUDE.md#environment-variables.

### Core

| Variable | Required | Default | Description |
|---|---|---|---|
| `NEXT_PUBLIC_WEBAPP_URL` | Yes | — | Public-facing URL (e.g., `https://app.documenso.com`) |
| `NEXT_PRIVATE_DATABASE_URL` | Yes | — | PostgreSQL connection string (pooled) |
| `NEXT_PRIVATE_DIRECT_DATABASE_URL` | Yes | — | PostgreSQL direct connection (for migrations) |
| `NEXTAUTH_SECRET` | Yes | — | Session encryption key (random 32+ bytes) |
| `NODE_ENV` | No | `development` | `development` / `production` / `test` |
| `PORT` | No | `3000` | Server port |

### Authentication

| Variable | Required | Default | Description |
|---|---|---|---|
| `NEXT_PRIVATE_GOOGLE_CLIENT_ID` | No | — | Google OAuth client ID |
| `NEXT_PRIVATE_GOOGLE_CLIENT_SECRET` | No | — | Google OAuth client secret |
| `NEXT_PRIVATE_OIDC_WELL_KNOWN` | No | — | Custom OIDC discovery URL |
| `NEXT_PRIVATE_OIDC_CLIENT_ID` | No | — | Custom OIDC client ID |
| `NEXT_PRIVATE_OIDC_CLIENT_SECRET` | No | — | Custom OIDC client secret |
| `NEXT_PRIVATE_DISABLE_SIGNUP` | No | `false` | Disable new user registration |

### Provider Selection

| Variable | Required | Default | Description |
|---|---|---|---|
| `NEXT_PRIVATE_UPLOAD_TRANSPORT` | No | `database` | Storage provider: `database` or `s3` |
| `NEXT_PRIVATE_SIGNING_TRANSPORT` | No | `local` | Signing provider: `local` or `gcloud-hsm` |
| `NEXT_PRIVATE_SMTP_TRANSPORT` | No | `smtp-auth` | Email provider: `smtp-auth`, `smtp-api`, `resend`, `mailchannels` |
| `NEXT_PRIVATE_JOBS_TRANSPORT` | No | `local` | Job queue provider: `local`, `bullmq`, `inngest` |

### Storage (S3)

Required when `NEXT_PRIVATE_UPLOAD_TRANSPORT=s3`.

| Variable | Required | Default | Description |
|---|---|---|---|
| `NEXT_PRIVATE_UPLOAD_BUCKET` | Conditional | — | S3 bucket name |
| `NEXT_PRIVATE_UPLOAD_REGION` | Conditional | `us-east-1` | S3 region |
| `NEXT_PRIVATE_UPLOAD_ENDPOINT` | No | AWS default | Custom S3 endpoint (for MinIO, R2, DigitalOcean Spaces) |
| `NEXT_PRIVATE_UPLOAD_ACCESS_KEY_ID` | Conditional | — | S3 access key |
| `NEXT_PRIVATE_UPLOAD_SECRET_ACCESS_KEY` | Conditional | — | S3 secret key |
| `NEXT_PRIVATE_UPLOAD_FORCE_PATH_STYLE` | No | `false` | Use path-style URLs (required for MinIO) |

### Signing (Local P12)

Required when `NEXT_PRIVATE_SIGNING_TRANSPORT=local`.

| Variable | Required | Default | Description |
|---|---|---|---|
| `NEXT_PRIVATE_SIGNING_LOCAL_FILE_PATH` | Conditional | — | Path to .p12 certificate file |
| `NEXT_PRIVATE_SIGNING_LOCAL_FILE_CONTENTS` | Conditional | — | Base64-encoded .p12 contents (alternative to file path) |
| `NEXT_PRIVATE_SIGNING_PASSPHRASE` | Conditional | — | Certificate passphrase |

**Note**: Provide either `FILE_PATH` or `FILE_CONTENTS`, not both.

### Signing (Google Cloud HSM)

Required when `NEXT_PRIVATE_SIGNING_TRANSPORT=gcloud-hsm`.

| Variable | Required | Default | Description |
|---|---|---|---|
| `NEXT_PRIVATE_SIGNING_GCLOUD_HSM_KEY_PATH` | Conditional | — | Full resource path to HSM key |
| `NEXT_PRIVATE_SIGNING_GCLOUD_APPLICATION_CREDENTIALS_CONTENTS` | Conditional | — | Base64-encoded GCP service account JSON |

### Email (SMTP Auth)

Required when `NEXT_PRIVATE_SMTP_TRANSPORT=smtp-auth`.

| Variable | Required | Default | Description |
|---|---|---|---|
| `NEXT_PRIVATE_SMTP_HOST` | Conditional | — | SMTP server hostname |
| `NEXT_PRIVATE_SMTP_PORT` | Conditional | `587` | SMTP port |
| `NEXT_PRIVATE_SMTP_USERNAME` | Conditional | — | SMTP username |
| `NEXT_PRIVATE_SMTP_PASSWORD` | Conditional | — | SMTP password |
| `NEXT_PRIVATE_SMTP_SECURE` | No | `false` | Use TLS (true for port 465) |
| `NEXT_PRIVATE_SMTP_FROM_ADDRESS` | Yes | — | Sender email address |
| `NEXT_PRIVATE_SMTP_FROM_NAME` | No | `Documenso` | Sender display name |

### Email (SMTP API)

Required when `NEXT_PRIVATE_SMTP_TRANSPORT=smtp-api`.

| Variable | Required | Default | Description |
|---|---|---|---|
| `NEXT_PRIVATE_SMTP_HOST` | Conditional | — | SMTP server hostname |
| `NEXT_PRIVATE_SMTP_PORT` | Conditional | `587` | SMTP port |
| `NEXT_PRIVATE_SMTP_APIKEY_HEADER` | Conditional | — | Header name for API key |
| `NEXT_PRIVATE_SMTP_APIKEY` | Conditional | — | API key value |

### Email (Resend)

Required when `NEXT_PRIVATE_SMTP_TRANSPORT=resend`.

| Variable | Required | Default | Description |
|---|---|---|---|
| `NEXT_PRIVATE_RESEND_API_KEY` | Conditional | — | Resend API key |

### Email (MailChannels)

Required when `NEXT_PRIVATE_SMTP_TRANSPORT=mailchannels`.

| Variable | Required | Default | Description |
|---|---|---|---|
| `NEXT_PRIVATE_MAILCHANNELS_API_KEY` | Conditional | — | MailChannels API key |
| `NEXT_PRIVATE_MAILCHANNELS_ENDPOINT` | No | default | Custom API endpoint |

### Jobs (BullMQ)

Required when `NEXT_PRIVATE_JOBS_TRANSPORT=bullmq`.

| Variable | Required | Default | Description |
|---|---|---|---|
| `NEXT_PRIVATE_REDIS_URL` | Conditional | — | Redis connection string |
| `NEXT_PRIVATE_REDIS_MAX_RETRIES` | No | `3` | Redis connection retry count |

### Jobs (Inngest)

Required when `NEXT_PRIVATE_JOBS_TRANSPORT=inngest`.

| Variable | Required | Default | Description |
|---|---|---|---|
| `INNGEST_EVENT_KEY` | Conditional | — | Inngest event key |
| `INNGEST_SIGNING_KEY` | Conditional | — | Inngest signing key |
| `INNGEST_BASE_URL` | No | — | Custom Inngest server URL (for self-hosted Inngest) |

### Billing (Stripe)

| Variable | Required | Default | Description |
|---|---|---|---|
| `NEXT_PRIVATE_STRIPE_API_KEY` | No | — | Stripe secret key |
| `NEXT_PRIVATE_STRIPE_WEBHOOK_SECRET` | No | — | Stripe webhook signing secret |
| `NEXT_PUBLIC_STRIPE_COMMUNITY_PLAN_MONTHLY_PRICE_ID` | No | — | Stripe price ID for community plan |
| `NEXT_PUBLIC_STRIPE_ENTERPRISE_PLAN_MONTHLY_PRICE_ID` | No | — | Stripe price ID for enterprise plan |

### Captcha (Cloudflare Turnstile)

| Variable | Required | Default | Description |
|---|---|---|---|
| `NEXT_PUBLIC_TURNSTILE_SITE_KEY` | No | — | Turnstile site key (enables captcha when set) |
| `NEXT_PRIVATE_TURNSTILE_SECRET_KEY` | No | — | Turnstile secret key |

### Observability

| Variable | Required | Default | Description |
|---|---|---|---|
| `NEXT_PUBLIC_POSTHOG_KEY` | No | — | PostHog project API key (enables analytics when set) |
| `NEXT_PRIVATE_DATADOG_API_KEY` | No | — | Datadog API key (enables APM when set) |

### Security

| Variable | Required | Default | Description |
|---|---|---|---|
| `NEXT_PRIVATE_ALLOWED_ORIGINS` | No | `NEXT_PUBLIC_WEBAPP_URL` | CORS allowed origins (comma-separated) |
| `NEXT_PRIVATE_EMBED_ALLOWED_ORIGINS` | No | — | Origins allowed to embed signing views in iframes |

---

## Provider Matrix

### Valid Combinations

| Storage | Signing | Email | Jobs | Best For |
|---|---|---|---|---|
| `database` | `local` | `smtp-auth` | `local` | **Quick Start**: Minimal dependencies. Single PostgreSQL instance |
| `s3` | `local` | `resend` | `local` | **Small Production**: S3 for documents, Resend for reliable email |
| `s3` | `local` | `smtp-auth` | `bullmq` | **Medium Production**: Redis for reliable jobs, own SMTP |
| `s3` | `gcloud-hsm` | `resend` | `bullmq` | **Enterprise/Cloud**: HSM signing, managed email, Redis jobs |
| `s3` | `gcloud-hsm` | `resend` | `inngest` | **Serverless**: Event-driven jobs, no Redis needed |

### Known Conflicts and Constraints

| Combination | Issue | Resolution |
|---|---|---|
| `database` storage + high volume (>1000 docs) | PDF blobs in PostgreSQL degrade query performance | Switch to `s3` storage |
| `local` jobs + multiple instances | Database polling has no distributed lock. Two workers may claim same job | Switch to `bullmq` or `inngest` for multi-instance deploys |
| `gcloud-hsm` + no GCP credentials | Startup fails with credential error | Provide `GCLOUD_APPLICATION_CREDENTIALS_CONTENTS` or use `local` signing |
| `inngest` + self-hosted without Inngest server | Jobs silently fail | Either host Inngest server or use `bullmq`/`local` |
| `mailchannels` + non-Cloudflare hosting | MailChannels requires Cloudflare Workers context | Use `resend` or `smtp-auth` outside Cloudflare |

---

## Feature Flags

Feature flags are controlled via environment variables and/or PostHog feature flags.

### Environment-Based Flags

| Flag | Default | Description |
|---|---|---|
| `NEXT_PUBLIC_AI_FEATURES_ENABLED` | `false` | Enable AI-assisted field placement. See @HARNESS.md |
| `NEXT_PUBLIC_DIRECT_TEMPLATES_ENABLED` | `true` | Enable direct link templates |
| `NEXT_PUBLIC_EMBEDDING_ENABLED` | `false` | Enable enterprise embedding feature |
| `NEXT_PRIVATE_DISABLE_SIGNUP` | `false` | Disable new user registration (self-hosted admin control) |
| `NEXT_PUBLIC_POSTHOG_KEY` | — | When set, enables PostHog analytics + feature flags |
| `NEXT_PUBLIC_TURNSTILE_SITE_KEY` | — | When set, enables captcha on public forms |

### PostHog Feature Flags (Runtime)

When PostHog is configured, additional flags can be toggled without deployment:

- Gradual rollout of new features
- A/B testing for UI changes
- Kill switches for problematic features

---

## Default Development Configuration

Minimal `.env.local` for local development:

```bash
# Core
NEXT_PUBLIC_WEBAPP_URL=http://localhost:3000
NEXT_PRIVATE_DATABASE_URL=postgresql://documenso:documenso@localhost:54320/documenso
NEXT_PRIVATE_DIRECT_DATABASE_URL=postgresql://documenso:documenso@localhost:54320/documenso
NEXTAUTH_SECRET=secret-for-dev-only-change-in-production

# Email (Inbucket — catches all emails locally)
NEXT_PRIVATE_SMTP_TRANSPORT=smtp-auth
NEXT_PRIVATE_SMTP_HOST=localhost
NEXT_PRIVATE_SMTP_PORT=2500
NEXT_PRIVATE_SMTP_FROM_ADDRESS=noreply@documenso.local
NEXT_PRIVATE_SMTP_SECURE=false

# Signing (development certificate)
NEXT_PRIVATE_SIGNING_TRANSPORT=local
NEXT_PRIVATE_SIGNING_LOCAL_FILE_CONTENTS=<base64-of-dev-cert>
NEXT_PRIVATE_SIGNING_PASSPHRASE=documenso

# Storage (database — simplest for dev)
NEXT_PRIVATE_UPLOAD_TRANSPORT=database

# Jobs (local — no Redis needed)
NEXT_PRIVATE_JOBS_TRANSPORT=local
```

Start dev services first:

```bash
docker compose up -d   # PostgreSQL :54320, Inbucket :9000/:2500, MinIO :9001/:9002
npm run dev            # Start application
```

See @CLAUDE.md#dev-services for service details.

## Cross-References

- Provider interface definitions: @INFRA.md#swappable-providers
- Docker compose services: @INFRA.md#development-docker-compose
- AI feature configuration: @HARNESS.md
- Security-related config: @SECURITY.md
- Job queue behavior per provider: @RUNTIME.md
- Zod validation in code: @CLAUDE.md#environment-variables
